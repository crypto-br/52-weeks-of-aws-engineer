# AWS Engineer Study Plan / 52 Sessions

Practical study guides for engineers with cloud experience who want to master AWS in depth. Each session is ~60 minutes and includes ASCII diagrams, CDK Python examples, application code, annotated CLI, documented pitfalls, and a reflection exercise.

**Target audience:** engineers with cloud/infra/Linux experience. This is not an introductory course — it assumes you already know what a VPC is, what an IAM role is, and how a container works.

---
## What each guide contains

All 52 guides follow the same structure:

```
Concepts           → markers [FACT] / [CONSENSUS] / [OPINION] / [UNCERTAIN]
ASCII Diagrams     → flows and architectures without image dependencies
CDK Python         → infrastructure as code, from scratch
Python application → SDK, clients, production utilities
Annotated CLI      → end-to-end operations with context
Pitfalls           → common mistakes and why they happen
Exercise           → open-ended questions to consolidate learning
References         → links to official primary documentation
```

---
## Index by Module

### Module 1: IaC Fundamentals

| # | Guide | Topic |
|---|-------|-------|
| 001 | [Advanced AWS CLI](EN/session-001-aws-cli-sso-assume-role.md) | SSO, named profiles, assume-role, pagination and JMESPath |
| 002 | [CloudFormation — stacks and templates](EN/session-002-cloudformation-stacks-templates.md) | Parameters, Ref/GetAtt, Outputs, deploy with changesets |
| 003 | [CloudFormation — changesets and drift](EN/session-003-cloudformation-changesets-drift.md) | Changesets, drift detection, stack policies |

### Module 2: CDK v2

| # | Guide | Topic |
|---|-------|-------|
| 004 | [CDK setup and bootstrap](EN/session-004-cdk-v2-setup-bootstrap.md) | Initialization, bootstrap, synth/diff/deploy |
| 005 | [Constructs L1, L2, L3](EN/session-005-cdk-constructs-l1-l2-l3.md) | When to use each level, Construct Hub, trade-offs |
| 006 | [Stacks and multi-account](EN/session-006-cdk-stacks-environments-multi-account.md) | Environments, multiple accounts, nested stacks |
| 007 | [Assets and bundling](EN/session-007-cdk-assets-bundling-lambda.md) | Lambda bundling, Docker images, local files |
| 008 | [Testing with assertions](EN/session-008-cdk-testing-assertions.md) | Fine-grained assertions, snapshot tests |
| 009 | [CDK Pipelines — bootstrap and OIDC](EN/session-009-cdk-pipelines-bootstrap-oidc.md) | Cross-account bootstrap, OIDC, pipeline structure |
| 010 | [CDK Pipelines — Stages and ShellSteps](EN/session-010-cdk-pipelines-stages-shellsteps.md) | Custom stages, ShellSteps, self-mutation |
| 011 | [CustomResources and Aspects](EN/session-011-cdk-custom-resources-aspects.md) | Custom Resources with Lambda, Aspects for global validation |
| 012 | [Context and feature flags](EN/session-012-cdk-context-feature-flags.md) | cdk.json, context lookups, feature flags for production |

### Module 3: ECS

| # | Guide | Topic |
|---|-------|-------|
| 013 | [Task Definitions](EN/session-013-ecs-task-definitions-logging.md) | Containers, volumes, logging, resource limits |
| 014 | [Services and ALB](EN/session-014-ecs-services-discovery-alb.md) | Services, service discovery, ALB integration |
| 015 | [Fargate networking and IAM](EN/session-015-ecs-fargate-networking-iam.md) | Networking, security groups, Task Role vs Execution Role |
| 016 | [Capacity Providers and autoscaling](EN/session-016-ecs-capacity-providers-autoscaling.md) | Capacity Providers, Application Auto Scaling |
| 017 | [Deploy strategies](EN/session-017-ecs-deploy-rolling-blue-green.md) | Rolling update, blue/green with CodeDeploy |
| 018 | [ECS Observability](EN/session-018-ecs-observabilidade-firelens-xray.md) | FireLens, Container Insights, X-Ray sidecar |

### Module 4: Lambda

| # | Guide | Topic |
|---|-------|-------|
| 019 | [Execution model and cold starts](EN/session-019-lambda-execution-cold-starts.md) | Init phases, cold starts, provisioned concurrency |
| 020 | [Event source mappings](EN/session-020-lambda-event-source-mappings.md) | SQS, Kinesis and DynamoDB Streams with filtering |
| 021 | [Extensions, Layers and Powertools](EN/session-021-lambda-extensions-layers-powertools.md) | Lambda Extensions, Layers, AWS Lambda Powertools |
| 024 | [Lambda Observability](EN/session-024-lambda-observabilidade-xray-insights.md) | Structured logging, X-Ray, Lambda Insights |
| 025 | [Lambda@Edge vs CloudFront Functions](EN/session-025-lambda-edge-cloudfront-functions.md) | Use cases, limits, when to choose each approach |

### Module 5: Orchestration with Step Functions

| # | Guide | Topic |
|---|-------|-------|
| 022 | [Standard vs Express and basic states](EN/session-022-stepfunctions-standard-express-states.md) | Task, Choice, Wait, Pass — pricing and semantic differences |
| 023 | [Parallel, Map and error handling](EN/session-023-stepfunctions-parallel-map-error.md) | Parallelism, array iteration, retry/catch |

### Module 6: Load Balancers

| # | Guide | Topic |
|---|-------|-------|
| 026 | [ALB — advanced listener rules](EN/session-026-alb-listener-rules-weighted.md) | Complex listener rules, weighted routing, fixed responses |
| 027 | [ALB — OIDC, mTLS and WAF](EN/session-027-alb-oidc-mtls-waf.md) | Native OIDC authentication, mutual TLS, WAF integration |
| 028 | [NLB and GLB](EN/session-028-nlb-glb-casos-uso.md) | NLB (TLS passthrough), GLB (inline inspection) |

### Module 7: DynamoDB

| # | Guide | Topic |
|---|-------|-------|
| 029 | [Access patterns first and PK/SK](EN/session-029-dynamodb-access-patterns-pk-sk.md) | Access pattern-oriented modeling, generic PK/SK |
| 030 | [Single-table design](EN/session-030-dynamodb-single-table-adjacency.md) | Adjacency list, overloaded indexes, entities in one table |
| 031 | [GSIs and LSIs](EN/session-031-dynamodb-gsi-lsi-hot-partitions.md) | Hot partitions, write amplification, GSI costs |
| 032 | [Streams and CDC](EN/session-032-dynamodb-streams-lambda-cdc.md) | DynamoDB Streams with Lambda, Change Data Capture |
| 033 | [DAX](EN/session-033-dax-arquitetura-casos-uso.md) | DAX architecture, use cases and when NOT to use |
| 034 | [Transactions and conditional operations](EN/session-034-dynamodb-transacoes-condicionais.md) | TransactWriteItems, ConditionExpression, guarantees and limits |
| 035 | [Global Tables](EN/session-035-dynamodb-global-tables-consistencia.md) | Multi-region eventual consistency, conflict resolution |

### Module 8: Observability

| # | Guide | Topic |
|---|-------|-------|
| 036 | [CloudWatch Custom Metrics (EMF)](EN/session-036-cloudwatch-custom-metrics-emf.md) | Embedded Metrics Format, custom metrics without PutMetricData cost |
| 037 | [CloudWatch Logs Insights](EN/session-037-cloudwatch-logs-insights-queries.md) | Query syntax, derived fields, dashboards |
| 038 | [Composite Alarms and anomaly detection](EN/session-038-cloudwatch-composite-alarms-anomaly.md) | Composite alarms, anomaly detection, automated actions |
| 039 | [Advanced X-Ray](EN/session-039-xray-groups-annotations-sampling.md) | Groups, annotations, sampling rules, cross-service integration |

### Module 9: FinOps

| # | Guide | Topic |
|---|-------|-------|
| 040 | [Savings Plans vs Reserved Instances](EN/session-040-finops-savings-plans-reserved.md) | Differences, flexibility, when to use each model |
| 041 | [Spot Instances and Spot Fleet](EN/session-041-finops-spot-instances-fleet.md) | Interruption handling, diversification, Spot Fleet strategies |
| 042 | [Cost Explorer and Compute Optimizer](EN/session-042-cost-explorer-anomaly-optimizer.md) | Cost Explorer, anomaly detection, instance rightsizing |
| 043 | [Budgets and tagging strategy](EN/session-043-budgets-acoes-automaticas-tagging.md) | Budgets with automatic actions, tagging for chargeback |

### Module 10: Secrets and Parameters

| # | Guide | Topic |
|---|-------|-------|
| 044 | [Secrets Manager](EN/session-044-secrets-manager-rotacao-rds.md) | Automatic rotation, Lambda rotators, RDS integration |
| 045 | [SSM Parameter Store](EN/session-045-ssm-parameter-store-hierarquias.md) | Hierarchies, SecureString, cost vs Secrets Manager |

### Module 11: EKS

| # | Guide | Topic |
|---|-------|-------|
| 046 | [Cluster provisioning and VPC CNI](EN/session-046-eks-cluster-provisioning-cni.md) | eksctl, VPC CNI networking, authentication modes |
| 047 | [Node groups and Fargate](EN/session-047-eks-node-groups-fargate.md) | Managed node groups vs Fargate profiles, upgrades, node drain |
| 048 | [IRSA and Pod Identity](EN/session-048-eks-irsa-pod-identity.md) | IAM for workloads without static credentials — two models compared |
| 049 | [Managed add-ons](EN/session-049-eks-addons-vpc-cni-ebs.md) | VPC CNI (Prefix Delegation), CoreDNS, EBS CSI Driver |
| 050 | [Karpenter](EN/session-050-eks-karpenter-node-provisioning.md) | Dynamic node provisioning, NodePool, EC2NodeClass |

### Module 12: Managed Databases

| # | Guide | Topic |
|---|-------|-------|
| 051 | [RDS — HA and proxy](EN/session-051-rds-multi-az-proxy-insights.md) | Multi-AZ, read replicas, RDS Proxy (pooling + IAM auth), Performance Insights |
| 052 | [Aurora — serverless and global](EN/session-052-aurora-serverless-global-database.md) | Aurora serverless (ACUs, auto-pause), Global Database, differences vs RDS |

---
## Technologies Covered

| Area | Services |
|------|----------|
| IaC | AWS CLI, CloudFormation, CDK v2 (Python) |
| Compute | ECS Fargate, Lambda, Lambda@Edge, CloudFront Functions |
| Orchestration | Step Functions (Standard + Express) |
| Containers | EKS, managed node groups, Fargate profiles, Karpenter |
| Networking | ALB, NLB, GLB, VPC CNI |
| Databases | DynamoDB, DAX, RDS Multi-AZ, RDS Proxy, Aurora serverless, Aurora Global Database |
| Observability | CloudWatch (EMF, Logs Insights, Composite Alarms, Anomaly Detection), X-Ray |
| Security | IRSA, Pod Identity, Secrets Manager, SSM Parameter Store, WAF, mTLS, OIDC |
| FinOps | Savings Plans, Reserved Instances, Spot Fleet, Cost Explorer, Budgets |
