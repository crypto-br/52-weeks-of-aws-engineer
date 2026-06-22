# Session — ECS: Capacity Providers and Application Auto Scaling

**Estimated duration:** 60 minutes
**Prerequisites:** session-015-ecs-fargate-networking-iam

---

## Objective

By the end, you will be able to configure a Capacity Provider for Fargate and Fargate Spot with weights, scale an ECS Service based on custom metrics (not just CPU/memory), and calculate the cost savings of Fargate Spot vs standard Fargate in a workload scenario with interruption tolerance.

---

## Context

[FACT] Capacity Providers are ECS's abstraction layer that separates the definition of *where* to run tasks (which capacity pool) from the definition of *what* to run (task definition). Before Capacity Providers, you specified `launchType: FARGATE` or `launchType: EC2` directly on the service — a binary and static choice. With Capacity Providers, you define a **strategy** with multiple providers and weights, and ECS distributes tasks among them automatically.

[FACT] Application Auto Scaling is the AWS service that manages scaling of ECS Services (among other resources like DynamoDB, Aurora, etc.). It's separate from EC2 Auto Scaling: while EC2 Auto Scaling manages instances, Application Auto Scaling manages the `desiredCount` of an ECS Service. Both can coexist in an EC2-based architecture (Application Auto Scaling increases tasks, EC2 Auto Scaling via Capacity Provider increases instances to accommodate them).

[CONSENSUS] The Fargate + Fargate Spot pair is the most commonly used combination for web services that tolerate some interruption. The typical production strategy maintains a base of standard Fargate tasks (availability guarantee) and scales predominantly on Fargate Spot (cost savings), using weights to control the proportion.

---

## Key Concepts

### 1. Capacity Providers: base and weight

[FACT] A service's Capacity Provider strategy has two parameters per provider:

- **`base`**: absolute minimum number of tasks that must run on this provider. Only one provider per strategy can have base > 0. It's satisfied before any weight calculation.
- **`weight`**: relative weight. After satisfying the `base`, additional tasks are distributed proportionally to the weights.

```
Example strategy:
  FARGATE:       base=2, weight=1
  FARGATE_SPOT:  base=0, weight=4

desiredCount=2:
  → FARGATE base is satisfied with 2 tasks
  → no remaining tasks to distribute by weight
  Result: 2 tasks in FARGATE, 0 in FARGATE_SPOT

desiredCount=7:
  → base: 2 tasks in FARGATE (base=2)
  → 5 remaining tasks to distribute by weight 1:4
    → 1 task in FARGATE  (1/5 × 5 = 1)
    → 4 tasks in FARGATE_SPOT  (4/5 × 5 = 4)
  Result: 3 tasks in FARGATE, 4 tasks in FARGATE_SPOT

desiredCount=10:
  → base: 2 tasks in FARGATE
  → 8 remaining tasks by weight 1:4
    → 1.6 → rounds to 2 in FARGATE
    → 6.4 → rounds to 6 in FARGATE_SPOT
  Result: ~4 tasks in FARGATE, ~6 tasks in FARGATE_SPOT
```

[FACT] The `base` guarantees that even when desiredCount is low (e.g., 1 task at midnight), you always have at least N tasks on stable capacity. This is critical for services that cannot tolerate zero availability during a Fargate Spot interruption event.

---

### 2. Fargate Spot: price, interruption, and graceful shutdown

[FACT] Fargate Spot runs tasks on surplus Fargate capacity at a reduced price compared to standard Fargate. The historically observed discount is approximately **70% relative to Fargate on-demand pricing** (the exact value varies by region and supply/demand fluctuation — AWS doesn't publish the percentage like it does with EC2 Spot).

[FACT] When AWS needs to reclaim Fargate Spot capacity, it issues a **2-minute** interruption warning before terminating the task. The warning arrives through two simultaneous channels:

1. **EventBridge**: an `ECS Task State Change` event with `stopCode: SpotInterruption` is published.
2. **SIGTERM** to the container: the main process receives SIGTERM, same as a normal shutdown.

```
t=0:   AWS decides to reclaim capacity
t=0:   EventBridge publishes SpotInterruption event
t=0:   Container receives SIGTERM
t=120: Container receives SIGKILL (if still running)
       Task is terminated
```

[FACT] The `stopTimeout` parameter in the container definition controls how long ECS waits between SIGTERM and SIGKILL. The default is 30 seconds. For Fargate Spot, you can configure up to 120 seconds (the 2-minute warning limit). Setting `stopTimeout: 120` gives the container maximum time to:
- Finish in-progress processing.
- Checkpoint state.
- Return SQS messages to the queue (nack/visibility timeout).
- Deregister from the service registry.

**Ideal workloads for Fargate Spot:**

```
✅ Suitable for Spot:
  - SQS queue processing (messages return to queue on SIGTERM)
  - Batch jobs with checkpoint (restarts from checkpoint)
  - Rendering, transcoding (job is reprocessed)
  - Development/staging environments
  - Stateless workers with guaranteed idempotency

❌ Unsuitable for Spot (use standard Fargate):
  - Stateful databases
  - Transactional processing without idempotency
  - WebSockets with critical connection state
  - Tasks that don't handle SIGTERM
```

---

### 3. Application Auto Scaling for ECS

[FACT] Application Auto Scaling manages the `desiredCount` of an ECS Service. It requires three components:

1. **Scalable Target**: registers the service as a scaling target.
2. **Scaling Policy**: defines *how* to scale (target tracking or step scaling).
3. **CloudWatch Alarm**: (automatic for target tracking, manual for step scaling).

**Target Tracking vs Step Scaling:**

```
Target Tracking:
  → You define a TARGET VALUE for a metric
  → AS calculates how many tasks are needed to maintain that value
  → Scales out when metric > target, in when metric < target
  → Simpler, recommended for most cases
  → Default scale-in cooldown: 300 seconds

Step Scaling:
  → You define THRESHOLDS and how many tasks to add/remove per threshold
  → More control, more complex to configure correctly
  → Recommended when you need rapid reaction to spikes
  → Can stack multiple steps (e.g., +2 tasks when CPU > 60%,
    +5 tasks when CPU > 80%)
```

[FACT] Predefined metrics available for ECS target tracking:

| Metric | Description | Typical target value |
|---|---|---|
| `ECSServiceAverageCPUUtilization` | Average task CPU | 50-70% |
| `ECSServiceAverageMemoryUtilization` | Average memory | 60-80% |
| `ALBRequestCountPerTarget` | Requests/second per task via ALB | Depends on app capacity |

---

### 4. Scaling by custom metrics: the SQS case

[FACT] The most robust pattern for queue processing services is to scale based on **backlog per task** — not the absolute number of messages in the queue, but the ratio between pending messages and processing tasks. This avoids the under-scaling problem when desiredCount is high and over-scaling when it's low.

```
Backlog per task formula:
  backlog_per_task = ApproximateNumberOfMessages / desiredCount

Example:
  ApproximateNumberOfMessages = 1000 messages in queue
  desiredCount = 5 tasks
  backlog_per_task = 200 messages/task

  If target is 100 messages/task:
  → To process 1000 messages with 100/task each
  → We need 1000/100 = 10 tasks
  → AS scales from 5 to 10
```

[FACT] To implement this formula in Application Auto Scaling, you use **Metric Math** in Target Tracking:

```json
{
  "CustomizedMetricSpecification": {
    "Metrics": [
      {
        "Id": "messages",
        "MetricStat": {
          "Metric": {
            "Namespace": "AWS/SQS",
            "MetricName": "ApproximateNumberOfMessagesVisible",
            "Dimensions": [{ "Name": "QueueName", "Value": "my-queue" }]
          },
          "Stat": "Sum"
        }
      },
      {
        "Id": "tasks",
        "MetricStat": {
          "Metric": {
            "Namespace": "ECS/ContainerInsights",
            "MetricName": "RunningTaskCount",
            "Dimensions": [
              { "Name": "ClusterName", "Value": "my-cluster" },
              { "Name": "ServiceName", "Value": "my-worker" }
            ]
          },
          "Stat": "Average"
        }
      },
      {
        "Id": "backlog",
        "Expression": "messages / tasks",
        "ReturnData": true
      }
    ]
  },
  "TargetValue": 100.0
}
```

---

## Practical Example

**Scenario:** An image processing service (`image-processor`) that:
- Consumes from an SQS queue.
- Tolerates interruptions (idempotent jobs with SQS visibility timeout as retry mechanism).
- Must maintain at least 2 on-demand tasks to guarantee minimum processing.
- Should scale predominantly on Fargate Spot for cost savings.
- Should scale based on backlog per task (target: 50 messages/task).

### Complete CDK (TypeScript)

```typescript
import { Stack, StackProps, Duration } from 'aws-cdk-lib';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as sqs from 'aws-cdk-lib/aws-sqs';
import * as appscaling from 'aws-cdk-lib/aws-applicationautoscaling';
import * as cloudwatch from 'aws-cdk-lib/aws-cloudwatch';
import { Construct } from 'constructs';

export class ImageProcessorStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    const vpc = ec2.Vpc.fromLookup(this, 'Vpc', { vpcName: 'prod-vpc' });

    // Input SQS queue
    const queue = new sqs.Queue(this, 'ImageQueue', {
      queueName: 'image-processing-queue',
      visibilityTimeout: Duration.seconds(300),  // 5 min to process each job
    });

    // Cluster with Fargate and Fargate Spot Capacity Providers
    const cluster = new ecs.Cluster(this, 'Cluster', {
      vpc,
      enableFargateCapacityProviders: true,  // enables FARGATE and FARGATE_SPOT
    });

    // Task Definition
    const taskDef = new ecs.FargateTaskDefinition(this, 'TaskDef', {
      cpu: 1024,
      memoryLimitMiB: 2048,
    });

    taskDef.addContainer('processor', {
      image: ecs.ContainerImage.fromRegistry('my-org/image-processor:latest'),
      environment: { QUEUE_URL: queue.queueUrl },
      logging: ecs.LogDrivers.awsLogs({ streamPrefix: 'processor' }),
      // Maximum time for graceful shutdown on Spot
      stopTimeout: Duration.seconds(120),
    });

    // Permission to consume from the queue
    queue.grantConsumeMessages(taskDef.taskRole);

    // ECS Service with Capacity Provider strategy
    const service = new ecs.FargateService(this, 'Service', {
      cluster,
      taskDefinition: taskDef,
      desiredCount: 2,
      // Strategy: 2 fixed tasks on Fargate, scale on Fargate Spot
      capacityProviderStrategies: [
        {
          capacityProvider: 'FARGATE',
          base: 2,       // always 2 on-demand tasks
          weight: 1,     // 1 of 5 additional tasks on on-demand
        },
        {
          capacityProvider: 'FARGATE_SPOT',
          base: 0,
          weight: 4,     // 4 of 5 additional tasks on Spot (80%)
        },
      ],
      vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
    });

    // ─── Application Auto Scaling ──────────────────────────────────────────

    const scaling = service.autoScaleTaskCount({
      minCapacity: 2,
      maxCapacity: 20,
    });

    // Queue messages metric
    const messagesVisible = new cloudwatch.Metric({
      namespace: 'AWS/SQS',
      metricName: 'ApproximateNumberOfMessagesVisible',
      dimensionsMap: { QueueName: queue.queueName },
      statistic: 'Sum',
      period: Duration.minutes(1),
    });

    // Scale based on ALB Request Count (simple alternative for APIs)
    // For SQS, we use step scaling with the backlog metric
    scaling.scaleOnMetric('ScaleOnQueueDepth', {
      metric: messagesVisible,
      scalingSteps: [
        { upper: 0, change: -1 },     // empty queue: remove 1 task
        { lower: 100, change: +1 },   // > 100 messages: add 1 task
        { lower: 500, change: +3 },   // > 500 messages: add 3 tasks
        { lower: 1000, change: +5 },  // > 1000 messages: add 5 tasks
      ],
      adjustmentType: appscaling.AdjustmentType.CHANGE_IN_CAPACITY,
      cooldown: Duration.seconds(120),
    });

    // Conservative scale in: wait 5 minutes before removing tasks
    scaling.scaleOnCpuUtilization('ScaleOnCpu', {
      targetUtilizationPercent: 60,
      scaleInCooldown: Duration.seconds(300),
      scaleOutCooldown: Duration.seconds(60),
    });
  }
}
```

### Calculating cost savings: Fargate vs Fargate Spot

```
Scenario: service with average desiredCount of 10 tasks (1 vCPU, 2 GB each)

Fargate pricing (us-east-1, approximate reference in 2025):
  CPU:    $0.04048 per vCPU-hour
  Memory: $0.004445 per GB-hour

Cost per task/hour (standard Fargate):
  = (1 vCPU × $0.04048) + (2 GB × $0.004445)
  = $0.04048 + $0.00889
  = $0.04937/hour per task

Cost per task/hour (Fargate Spot, ~70% discount):
  = $0.04937 × 0.30
  = ~$0.01481/hour per task

Monthly cost with mixed strategy (base=2 Fargate, 8 Spot):
  Standard Fargate:  2 tasks × $0.04937 × 720h = $71.09
  Fargate Spot:      8 tasks × $0.01481 × 720h = $85.34
  Total:             $156.43/month

Monthly cost with 100% standard Fargate:
  10 tasks × $0.04937 × 720h = $355.46/month

Savings with the mixed strategy: ~56% (~$199/month)
```

[UNCERTAIN] Exact Fargate Spot prices vary by region and fluctuate with supply/demand. The 70% discount is based on commonly observed historical data, but may be different at the time you're reading. Check the current price at `aws.amazon.com/fargate/pricing`.

---

## Common Pitfalls

### Pitfall 1: base=0 on both providers — tasks always on Spot during reduced scaling

**The error:** You configure the strategy `FARGATE: base=0, weight=1` and `FARGATE_SPOT: base=0, weight=4`. With `desiredCount=1` (e.g., service under low load at night), the only task goes to Fargate Spot (higher weight). A Spot interruption at night takes down the service's only task. The ALB starts returning 503. ECS starts a new task on Spot, but while the new task comes up (~30-60s), the service is unavailable.

**Why it happens:** With `base=0`, ECS distributes *all* tasks by weight, including the first one. Without any task guaranteed on standard Fargate, a Spot interruption results in downtime during replacement time.

**How to avoid:** Always set `base >= 1` (or `base >= 2` for zero-downtime during replacement) on the most stable provider (standard FARGATE). The `base` is the minimum availability insurance.

---

### Pitfall 2: Short scale-in cooldown causing thrashing

**The error:** You configure target tracking with `scaleInCooldown: Duration.seconds(60)`. An SQS queue has burst processing: fills quickly, is processed in 2 minutes, empties. AS scales out to 10 tasks, the queue empties, AS scales in to 2 tasks in 60 seconds. Thirty seconds later, a new message arrives, AS scales out again. This cycle creates **thrashing** — tasks being created and destroyed every few minutes, which:
- Increases cost (minimum 1-minute charge per Fargate task).
- Degrades performance (task cold start time).
- Creates excessive noise in logs and metrics.

**Why it happens:** Fast scale-in without understanding the message arrival pattern. For queues with periodic bursts, the cooldown should be longer than the period between bursts.

**How to avoid:** For SQS queues, use `scaleInCooldown: Duration.seconds(300)` (5 minutes) or longer. Scale out can be fast (30-60s), but scale in should be conservative. Analyze the historical traffic pattern before configuring cooldowns.

---

### Pitfall 3: Not handling SIGTERM in Fargate Spot containers

**The error:** The worker container on Fargate Spot uses a Python process without signal handling. When the Spot interruption occurs, the process receives SIGTERM but doesn't handle it — it keeps running. After `stopTimeout` (default: 30 seconds), ECS sends SIGKILL. The SQS message being processed doesn't return to the queue (the worker didn't nack or reset the visibility timeout). The message goes to the DLQ after exceeding `maxReceiveCount`, or gets "lost" if processing was halfway done.

**Why it happens:** Most runtimes ignore SIGTERM by default. Python, Node.js, and Java need explicit SIGTERM handlers.

**How to avoid:** Implement a SIGTERM handler that:
1. Stops accepting new messages from the queue.
2. Waits for current processing to finish (or cancels and nacks).
3. Exits with `exit(0)`.

```python
# Python — SIGTERM handler example for SQS worker
import signal
import sys

shutdown_requested = False

def handle_sigterm(signum, frame):
    global shutdown_requested
    print("SIGTERM received — initiating graceful shutdown")
    shutdown_requested = True

signal.signal(signal.SIGTERM, handle_sigterm)

while not shutdown_requested:
    messages = sqs.receive_message(QueueUrl=QUEUE_URL, MaxNumberOfMessages=1)
    if 'Messages' in messages:
        msg = messages['Messages'][0]
        try:
            process(msg)
            sqs.delete_message(QueueUrl=QUEUE_URL, ReceiptHandle=msg['ReceiptHandle'])
        except Exception as e:
            # Don't delete — message returns to queue automatically
            print(f"Error: {e}")

print("Shutdown complete")
sys.exit(0)
```

---

## Reflection Exercise

You're sizing a video transcoding service on ECS Fargate that processes jobs from an SQS queue. Each job takes an average of 8 minutes to complete. Job volume has a clear daily pattern: peak of 500 jobs/hour from 2 PM to 6 PM and low of 10 jobs/hour from midnight to 6 AM.

How would you configure the Capacity Provider strategy (base and weight between Fargate and Fargate Spot)? What would be the appropriate `stopTimeout` for the container? Which scaling metric would be most appropriate — CPU, queue message count, or backlog per task — and what would be the target value? What are the risks of using Fargate Spot for this type of workload and how does the worker design (SIGTERM handling, idempotency, checkpoint) mitigate those risks? Finally, how would you calculate the estimated monthly cost with the mixed strategy versus 100% standard Fargate?

---

## Resources for Further Study

### 1. Amazon ECS clusters for Fargate (Capacity Providers)
**URL:** https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-capacity-providers.html
**What to find:** Complete documentation of FARGATE and FARGATE_SPOT Capacity Providers: how to configure the strategy, base and weight values, the Spot interruption mechanism, and how to handle SIGTERM.
**Why it's the right source:** It's the primary reference for Fargate Capacity Providers — includes the interruption protocol details that any Spot operator needs to understand.

### 2. Automatically scale your Amazon ECS service
**URL:** https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-auto-scaling.html
**What to find:** Complete guide to Application Auto Scaling for ECS: target tracking, step scaling, predefined and custom metrics, and cooldown configuration. Includes examples with AWS CLI.
**Why it's the right source:** It's the official guide that integrates ECS with Application Auto Scaling, covering both concepts and practical configuration examples.

### 3. Optimizing Amazon ECS service auto scaling
**URL:** https://docs.aws.amazon.com/AmazonECS/latest/developerguide/capacity-autoscaling-best-practice.html
**What to find:** Best practices guide specific to ECS scaling: how to choose the right metric by workload type, cooldown pitfalls, and the backlog-per-task pattern for SQS queues.
**Why it's the right source:** It's the prescriptive guide — it says *what to do* beyond *how to do it*, with justifications for each recommendation.

---
