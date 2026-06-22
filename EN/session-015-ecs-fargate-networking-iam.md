# Session 15 — ECS Fargate: networking, security groups, and IAM Roles for Tasks

**Estimated duration:** 60 minutes
**Prerequisites:** session-014-ecs-services-discovery-alb

---

## Objective

By the end, you will be able to explain the Fargate `awsvpc` networking model (each task has its own ENI), configure security groups at the task level, create a task IAM role with minimal permissions, and verify via `aws sts get-caller-identity` inside the container.

---

## Context

[FACT] Before the `awsvpc` network mode (introduced in 2017), ECS containers on EC2 shared the host's network interface. This meant security groups could only be applied at the EC2 instance level, not per container — a coarse model that made per-service network segmentation impossible without dedicated instances. Fargate, launched the same year, adopted `awsvpc` as the only available mode and made granular network isolation the default.

[FACT] `awsvpc` gives each ECS task the same network position as an EC2 instance: its own private IP, its own security groups, an AWS-managed ENI, and presence in the VPC routing table. From the network's perspective, a Fargate task is indistinguishable from a small EC2 instance.

[CONSENSUS] Fargate's credential model is often misunderstood by teams coming from traditional EC2 environments. On EC2, instance credentials arrive via IMDS (`169.254.169.254`). On Fargate and ECS with `awsvpc`, task credentials arrive via a different endpoint (`169.254.170.2`), managed by the ECS agent. Understanding this distinction is essential for debugging `AccessDenied` in containers and for configuring IAM correctly.

---

## Key Concepts

### 1. awsvpc: one ENI per task

[FACT] When ECS starts a Fargate task, it automatically provisions an **Elastic Network Interface (ENI)** in the specified subnet. This ENI receives:

- A primary private IP (from the subnet's CIDR range).
- The security groups specified in the service or task network configuration.
- Optionally, a public IP (if `assignPublicIp: ENABLED` and the subnet is public).

```
VPC (10.0.0.0/16)
│
├── Private Subnet A (10.0.1.0/24)
│   ├── ENI: 10.0.1.5  → Task 1 of api-service  [sg-api]
│   ├── ENI: 10.0.1.8  → Task 2 of api-service  [sg-api]
│   └── ENI: 10.0.1.12 → Task 1 of db-migrator  [sg-migrator]
│
└── Private Subnet B (10.0.2.0/24)
    ├── ENI: 10.0.2.3  → Task 3 of api-service  [sg-api]
    └── ENI: 10.0.2.7  → Task 1 of worker       [sg-worker]
```

[FACT] Important operational implications of the one-ENI-per-task model:

1. **ENI limit per account/region**: each EC2 instance type has an ENI limit per host, but in Fargate the relevant limit is the ENI limit per subnet/VPC. The default is 5,000 ENIs per VPC. For services with high auto scaling, this limit can be reached.

2. **VPC Flow Logs per task**: since each task has its own ENI, VPC Flow Logs records traffic at the task level. You can filter by IP to see exactly the traffic of a specific task — something impossible in the bridge/host model.

3. **Granular security groups**: the security group is assigned to the task's ENI, not the host. Two different services running in Fargate can have completely different security groups with no relationship between them.

4. **Containers in the same task share the ENI**: all containers within a task share the same network space (same ENI, same IP). Communication between containers in the same task is via `localhost`, not via the ENI IP.

---

### 2. Security groups at the task level

[FACT] Security groups in ECS Fargate work exactly like in EC2: stateful, with separate inbound and outbound rules, referenceable between each other. The difference is you apply them to the service/task instead of to the instance.

**Recommended model for microservices:**

```
┌─────────────┐    sg-alb         ┌─────────────┐
│     ALB     │──────────────────▶│  api-service│  sg-api
│  (sg-alb)   │                   │             │
└─────────────┘                   └──────┬──────┘
                                         │
                              sg-api → sg-db (5432)
                                         │
                                  ┌──────▼──────┐
                                  │  RDS Aurora │  sg-db
                                  └─────────────┘

Rules:
sg-api inbound:
  - port 3000 from sg-alb       (ALB talks to the task)

sg-api outbound:
  - port 443 to 0.0.0.0/0      (HTTPS to AWS APIs)
  - port 5432 to sg-db          (PostgreSQL to RDS)

sg-db inbound:
  - port 5432 from sg-api       (only api-service accesses the database)
```

[CONSENSUS] Referencing security groups between each other (instead of using CIDRs) is the correct approach in VPCs with many services. When you use `source: sg-api` in a `sg-db` rule, the rule automatically includes any new task from api-service without needing to update the CIDR. It's dynamic by definition.

[FACT] A critical outbound rule that's often forgotten: for tasks in **private subnets**, you need outbound access to call AWS APIs (ECR, CloudWatch, Secrets Manager, S3). The options are:

```
Option 1: NAT Gateway
  Task (private) → NAT GW (public) → Internet → AWS API
  Cost: ~$32/month fixed + $0.045/GB processed
  Simplicity: maximum (single egress point)

Option 2: VPC Endpoints (Interface + Gateway)
  Task (private) → VPC Endpoint ENI → AWS API (via PrivateLink)
  Cost: ~$7.30/month per endpoint per AZ + $0.01/GB
  Security: traffic never leaves the AWS network
  Complexity: need to create individual endpoints per service

Option 3: Public subnet with public IP
  Task (public, assignPublicIp=ENABLED) → Internet Gateway → AWS API
  Cost: minimal (IGW is free)
  Security: task directly exposed to the internet (avoid in production)
```

[FACT] For Fargate in private subnets, the minimum VPC endpoints needed to start a task are:

| Endpoint | Type | Purpose |
|---|---|---|
| `com.amazonaws.REGION.ecr.api` | Interface | Image manifest pull |
| `com.amazonaws.REGION.ecr.dkr` | Interface | Image layer pull |
| `com.amazonaws.REGION.s3` | Gateway | Layer download from S3 (ECR uses S3) |
| `com.amazonaws.REGION.logs` | Interface | CloudWatch Logs (awslogs driver) |
| `com.amazonaws.REGION.secretsmanager` | Interface | If using Secrets Manager |
| `com.amazonaws.REGION.ssm` | Interface | If using Parameter Store |

---

### 3. Task IAM Role: how credentials reach the container

[FACT] The flow of IAM credentials inside a Fargate container is managed by the ECS agent and works like this:

```
┌─────────────────────────────────────────────────────────────┐
│  Credential injection process (initiated by ECS)            │
│                                                             │
│  1. Task is started with taskRoleArn defined                │
│  2. ECS agent does AssumeRole on the Task Role              │
│     → obtains temporary credentials (AccessKey + Secret     │
│       + SessionToken), valid for 6 hours                    │
│  3. ECS agent stores credentials in internal cache          │
│  4. ECS agent sets in the container:                        │
│     AWS_CONTAINER_CREDENTIALS_RELATIVE_URI=/v2/credentials/TOKEN │
│                                                             │
│  5. Inside the container, the AWS SDK does GET to:          │
│     http://169.254.170.2/v2/credentials/TOKEN               │
│     → receives temporary credentials in JSON                │
│                                                             │
│  6. Every ~5 hours, the ECS agent renews automatically      │
└─────────────────────────────────────────────────────────────┘
```

[FACT] The `169.254.170.2` endpoint is a link-local address managed by the ECS agent, different from EC2's IMDS (`169.254.169.254`). ECS blocks access to EC2's IMDS from inside Fargate containers — containers cannot assume the underlying host's identity.

[FACT] You can verify which identity is being used inside a container with:

```bash
# Inside the container (via ECS Exec or during local tests):
aws sts get-caller-identity

# Expected output for a task with Task Role configured:
{
    "UserId": "AROAEXAMPLEID:TASK-ID",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/my-task-role/TASK-ID"
}

# If no Task Role is configured, the SDK returns an error:
# An error occurred (NoCredentialProviders): no valid credential providers found
```

[FACT] `ECS Exec` (`aws ecs execute-command`) allows opening an interactive shell inside a running container, which is very useful for debugging IAM:

```bash
aws ecs execute-command \
  --cluster my-cluster \
  --task <task-id> \
  --container api \
  --interactive \
  --command "/bin/bash"

# Inside the container shell:
$ aws sts get-caller-identity
$ curl -s http://169.254.170.2$AWS_CONTAINER_CREDENTIALS_RELATIVE_URI | python3 -m json.tool
```

For ECS Exec to work, the task definition needs `enableExecuteCommand: true` and the Task Role needs the `ssmmessages:CreateControlChannel` permission (among others).

---

### 4. Principle of least privilege for Task Roles

[CONSENSUS] In ECS, the most common anti-pattern is creating a single task role with `AdministratorAccess` or `PowerUserAccess` and reusing it across all services. This violates the principle of least privilege: if a container is compromised (code vulnerability, SSRF, etc.), the attacker gains unrestricted access to the AWS account.

[FACT] The least privilege policy for Task Roles requires:

1. **One Task Role per service** (not shared between services with different functions).
2. **Specific resources by ARN** — never `"Resource": "*"` when the ARN is known at deploy time.
3. **Condition keys** when available, to restrict further.

Example of a Task Role for a service that reads from a specific S3 bucket and writes to a specific DynamoDB:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::my-app-bucket/data/*"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::my-app-bucket",
      "Condition": {
        "StringLike": { "s3:prefix": ["data/*"] }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:UpdateItem",
        "dynamodb:Query"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/my-app-table"
    }
  ]
}
```

CDK equivalent (using `grant*` methods that generate minimal policies automatically):

```typescript
// CDK generates the correct minimal permissions for each operation
bucket.grantRead(taskRole, 'data/*');
table.grantReadWriteData(taskRole);

// For Secrets Manager with specific path:
const secret = secretsmanager.Secret.fromSecretNameV2(this, 'DbSecret', 'prod/db');
secret.grantRead(taskRole);
```

---

## Practical Example

**Complete scenario:** An `api-service` in Fargate running in a private subnet that needs:
- Receive traffic from the ALB on port 3000.
- Call DynamoDB and Secrets Manager.
- Pull image from private ECR.
- Send logs to CloudWatch.
- Be accessible for debugging via ECS Exec.

### Complete CDK

```typescript
import { Stack, StackProps } from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';
import * as secretsmanager from 'aws-cdk-lib/aws-secretsmanager';
import * as ecr from 'aws-cdk-lib/aws-ecr';
import * as logs from 'aws-cdk-lib/aws-logs';
import { Construct } from 'constructs';

export class ApiServiceStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    const vpc = ec2.Vpc.fromLookup(this, 'Vpc', { vpcName: 'prod-vpc' });

    // ─── VPC Endpoints (for private subnet without NAT) ──────────────────────
    // Gateway endpoint for S3 (free, needed for ECR)
    vpc.addGatewayEndpoint('S3Endpoint', {
      service: ec2.GatewayVpcEndpointAwsService.S3,
    });
    // Interface endpoints for ECR, CloudWatch Logs, Secrets Manager
    vpc.addInterfaceEndpoint('EcrApiEndpoint', {
      service: ec2.InterfaceVpcEndpointAwsService.ECR,
    });
    vpc.addInterfaceEndpoint('EcrDkrEndpoint', {
      service: ec2.InterfaceVpcEndpointAwsService.ECR_DOCKER,
    });
    vpc.addInterfaceEndpoint('LogsEndpoint', {
      service: ec2.InterfaceVpcEndpointAwsService.CLOUDWATCH_LOGS,
    });
    vpc.addInterfaceEndpoint('SecretsManagerEndpoint', {
      service: ec2.InterfaceVpcEndpointAwsService.SECRETS_MANAGER,
    });

    // ─── Security Groups ──────────────────────────────────────────────────

    // ALB SG (externally managed, referenced here)
    const albSg = ec2.SecurityGroup.fromLookupByName(this, 'AlbSg', 'sg-alb', vpc);

    // Task SG
    const taskSg = new ec2.SecurityGroup(this, 'TaskSg', {
      vpc,
      description: 'Security group for api-service tasks',
      allowAllOutbound: false,   // least privilege: explicit outbound
    });
    // Inbound: only from ALB
    taskSg.addIngressRule(albSg, ec2.Port.tcp(3000), 'Traffic from ALB');
    // Outbound: HTTPS to VPC endpoints and DynamoDB
    taskSg.addEgressRule(ec2.Peer.anyIpv4(), ec2.Port.tcp(443), 'HTTPS to AWS APIs');

    // ─── Task Role (application code identity) ────────────────────────────

    const taskRole = new iam.Role(this, 'TaskRole', {
      assumedBy: new iam.ServicePrincipal('ecs-tasks.amazonaws.com'),
      description: 'Task role for api-service — least privilege',
    });

    // Application permissions (using grant* to generate minimal policies)
    const table = dynamodb.Table.fromTableName(this, 'AppTable', 'api-app-table');
    table.grantReadWriteData(taskRole);

    const dbSecret = secretsmanager.Secret.fromSecretNameV2(
      this, 'DbSecret', 'prod/api/db-credentials'
    );
    dbSecret.grantRead(taskRole);

    // Permissions for ECS Exec (interactive debugging)
    taskRole.addToPolicy(new iam.PolicyStatement({
      actions: [
        'ssmmessages:CreateControlChannel',
        'ssmmessages:CreateDataChannel',
        'ssmmessages:OpenControlChannel',
        'ssmmessages:OpenDataChannel',
      ],
      resources: ['*'],
    }));

    // ─── Task Definition ──────────────────────────────────────────────────

    const logGroup = new logs.LogGroup(this, 'LogGroup', {
      logGroupName: '/ecs/api-service',
      retention: logs.RetentionDays.ONE_MONTH,
    });

    const repo = ecr.Repository.fromRepositoryName(this, 'Repo', 'api-service');

    const taskDef = new ecs.FargateTaskDefinition(this, 'TaskDef', {
      cpu: 512,
      memoryLimitMiB: 1024,
      taskRole,
      // executionRole created by CDK with permissions for ECR pull and CloudWatch logs
    });

    taskDef.addContainer('api', {
      image: ecs.ContainerImage.fromEcrRepository(repo, 'latest'),
      portMappings: [{ containerPort: 3000 }],
      logging: ecs.LogDrivers.awsLogs({
        streamPrefix: 'api',
        logGroup,
        mode: ecs.AwsLogDriverMode.NON_BLOCKING,
      }),
      secrets: {
        DB_PASSWORD: ecs.Secret.fromSecretsManager(dbSecret, 'password'),
        DB_HOST: ecs.Secret.fromSecretsManager(dbSecret, 'host'),
      },
    });

    // ─── ECS Service ──────────────────────────────────────────────────────

    const cluster = ecs.Cluster.fromClusterAttributes(this, 'Cluster', {
      clusterName: 'prod-cluster',
      vpc,
      securityGroups: [],
    });

    new ecs.FargateService(this, 'Service', {
      cluster,
      taskDefinition: taskDef,
      desiredCount: 3,
      securityGroups: [taskSg],
      // Private subnets — no public IP
      assignPublicIp: false,
      vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
      enableExecuteCommand: true,   // enables ECS Exec for debugging
      deploymentCircuitBreaker: { rollback: true },
    });
  }
}
```

### Verifying identity inside the container

```bash
# Open shell in running container
aws ecs execute-command \
  --cluster prod-cluster \
  --task arn:aws:ecs:us-east-1:123456789012:task/prod-cluster/abc123 \
  --container api \
  --interactive \
  --command "/bin/sh"

# Inside the container, verify the task role identity
$ aws sts get-caller-identity
{
    "UserId": "AROAEXAMPLE123:abc123",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/ApiServiceStack-TaskRole/abc123"
}

# Verify the environment variable injected by ECS
$ echo $AWS_CONTAINER_CREDENTIALS_RELATIVE_URI
/v2/credentials/a1b2c3d4-e5f6-7890-abcd-ef1234567890

# Inspect the raw credentials (useful for debugging)
$ curl -s http://169.254.170.2$AWS_CONTAINER_CREDENTIALS_RELATIVE_URI
{
  "RoleArn": "arn:aws:iam::123456789012:role/ApiServiceStack-TaskRole",
  "AccessKeyId": "ASIA...",
  "SecretAccessKey": "...",
  "Token": "...",
  "Expiration": "2026-06-03T18:00:00Z"
}

# Test DynamoDB access
$ aws dynamodb describe-table --table-name api-app-table --region us-east-1
```

---

## Common Pitfalls

### Pitfall 1: `allowAllOutbound: true` on the task security group (CDK default)

**The error:** CDK creates security groups with `allowAllOutbound: true` by default. For tasks in private subnets without VPC endpoints, this works if there's a NAT gateway — the task accesses any AWS endpoint via NAT. But when you remove the NAT gateway to save costs and add VPC endpoints, traffic to services without endpoints (e.g., STS, EC2 Describe, CloudFormation) keeps trying to go to the internet, fails silently, and the task can't start or function.

**Why it happens:** With `allowAllOutbound: true`, the security group allows outbound traffic, but without NAT and without an endpoint for the specific service, the traffic has nowhere to go in the VPC and is dropped by the routing table.

**How to recognize:** Tasks stuck in `PROVISIONING` state or failing with `CannotPullContainerError` indicating timeout on image pull from ECR. In VPC Flow Logs, you see REJECTED outbound traffic to AWS public IPs.

**How to avoid:** Use `allowAllOutbound: false` and add explicit outbound rules. Before removing a NAT gateway, audit which AWS endpoints the service calls and create corresponding VPC endpoints. The command `aws ec2 describe-vpc-endpoint-services` lists all services available as VPC endpoints in your region.

---

### Pitfall 2: Task Role without permission, but Execution Role with excess permissions

**The error:** A task needs to read from Secrets Manager. The developer doesn't configure `taskRoleArn` and instead adds `secretsmanager:GetSecretValue` to the Execution Role. The task starts and manages to inject the secret at initialization via the task definition's `secrets` (this works, since it's the ECS agent making the call). But the application code that tries to call `GetSecretValue` directly (e.g., to rotate credentials at runtime) fails with `AccessDenied`, because the credentials injected in the container are from the Task Role (empty), not the Execution Role.

**Why it happens:** The Execution Role is not injected into the container — it's used exclusively by the ECS agent before and during initialization. Once the container is running, only the Task Role is accessible via `AWS_CONTAINER_CREDENTIALS_RELATIVE_URI`.

**How to recognize:** The container starts successfully (secret injection at initialization works via Execution Role), but SDK calls inside the code fail with `NoCredentialProviders` or `AccessDenied` when trying to access any AWS service.

**How to avoid:** Definitive rule — every permission the **application code** needs at runtime goes in the **Task Role**. Every permission **ECS** needs to make the container born goes in the **Execution Role**.

---

### Pitfall 3: ENI limit reached during auto scaling

**The error:** You have a service with `desiredCount=10` that scales up to `maxCount=100` during peaks. Your VPC has 3 AZs and private subnets with /24 each (253 usable IPs). Aggressive auto scaling creates 100 tasks at once, each occupying an ENI in one of the subnets. With 100 tasks + other ENIs from existing services + ENIs from RDS instances, Lambda ENIs, etc., you can hit the subnet's available IP limit or the per-VPC ENI limit.

**Why it happens:** Unlike EC2 (where multiple containers share an ENI), in Fargate each task consumes one ENI with one IP from the subnet. /24 subnets with 253 IPs are limiting for high-scale services.

**How to recognize:** Scaling failures with `SERVICE DEPLOYMENT FAILED: Tasks failed to start` and `EniLimitExceeded` or `InsufficientPrivateIPAddressCapacity` in the ECS service events.

**How to avoid:** Size subnets for peak tasks × number of AZs with 2x margin. For services with potential for dozens of tasks, use /22 or larger subnets. Proactively monitor the CloudWatch metric `SubnetsAvailableIPAddressCount` with alarms before hitting the limit.

---

## Reflection Exercise

You're auditing the security of an existing ECS Fargate system and find the following configuration:

- All services use the same Task Role with the `PowerUserAccess` policy.
- All task security groups have `allowAllOutbound: true`.
- Services run in public subnets with `assignPublicIp: ENABLED`.
- Tasks don't have `enableExecuteCommand` configured.

How would you prioritize the fixes? For each problem, what is the real risk and what is the minimum change to mitigate it? When migrating to private subnets, which VPC endpoints would you create first to ensure tasks continue working? If you need to keep at least one operational environment during migration, what would be the sequence of changes?

---

## Resources for Further Study

### 1. Amazon ECS task networking (awsvpc mode)
**URL:** https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-networking-awsvpc.html
**What to find:** Complete documentation of awsvpc mode: how ENIs are provisioned, limits per EC2 instance (relevant for EC2 launch type), public vs private IP configuration, and integration with VPC Flow Logs.
**Why it's the right source:** It's the primary technical reference for the networking model, more detailed than the Fargate-specific guide.

### 2. Amazon ECS task IAM role
**URL:** https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-iam-roles.html
**What to find:** How to configure the Task Role, the credential injection mechanism via `169.254.170.2`, how to verify credentials inside the container, and the explicit difference between Task Role and Execution Role.
**Why it's the right source:** It's the canonical documentation of the IAM credential mechanism in ECS — the place to go when you have doubts about `AccessDenied` in containers.

### 3. Best practices for connecting Amazon ECS to AWS services from inside your VPC
**URL:** https://docs.aws.amazon.com/AmazonECS/latest/developerguide/networking-connecting-vpc.html
**What to find:** Detailed comparison between NAT Gateway, VPC Endpoints, and public subnet for accessing AWS services from ECS tasks. Includes cost tables and per-use-case recommendations.
**Why it's the right source:** It's the official best practices guide that addresses exactly the most common trade-off (NAT vs Endpoints), with cost data included.

---
