# Plano de Estudos AWS Enginner / 52 Sessões

Guias de estudo práticos para engenheiros com experiência em cloud que querem dominar AWS em profundidade. Cada sessão tem ~60 minutos e inclui diagramas ASCII, exemplos CDK Python, código de aplicação, CLI comentado, armadilhas documentadas e um exercício de reflexão.

**Público-alvo:** engenheiros com experiência em cloud/infra/Linux. Não é um curso introdutório parte do princípio que você já sabe o que é uma VPC, uma IAM role e como um container funciona.

---
## O que tem em cada guia

Todos os 52 guias seguem a mesma estrutura:

```
Conceitos          → marcadores [FATO] / [CONSENSO] / [OPINIÃO] / [INCERTO]
Diagramas ASCII    → fluxos e arquiteturas sem dependência de imagens
CDK Python         → infraestrutura como código, do zero
Python aplicação   → SDK, clientes, utilitários de produção
CLI comentado      → operações do início ao fim com contexto
Armadilhas         → erros comuns e por que acontecem
Exercício          → perguntas abertas para consolidar o aprendizado
Referências        → links para documentação primária oficial
```

---
## Índice por módulo

### Módulo 1: Fundamentos de IaC

| # | Guia | Tópico |
|---|------|--------|
| 001 | [AWS CLI avançado](session-001-aws-cli-sso-assume-role.md) | SSO, perfis nomeados, assume-role, paginação e JMESPath |
| 002 | [CloudFormation — stacks e templates](session-002-cloudformation-stacks-templates.md) | Parameters, Ref/GetAtt, Outputs, deploy com changesets |
| 003 | [CloudFormation — changesets e drift](session-003-cloudformation-changesets-drift.md) | Changesets, drift detection, stack policies |

### Módulo 2: CDK v2

| # | Guia | Tópico |
|---|------|--------|
| 004 | [CDK setup e bootstrap](session-004-cdk-v2-setup-bootstrap.md) | Inicialização, bootstrap, synth/diff/deploy |
| 005 | [Constructs L1, L2, L3](session-005-cdk-constructs-l1-l2-l3.md) | Quando usar cada nível, Construct Hub, trade-offs |
| 006 | [Stacks e multi-account](session-006-cdk-stacks-environments-multi-account.md) | Environments, múltiplas contas, stacks nested |
| 007 | [Assets e bundling](session-007-cdk-assets-bundling-lambda.md) | Lambda bundling, Docker images, arquivos locais |
| 008 | [Testes com assertions](session-008-cdk-testing-assertions.md) | Fine-grained assertions, snapshot tests |
| 009 | [CDK Pipelines — bootstrap e OIDC](session-009-cdk-pipelines-bootstrap-oidc.md) | Bootstrap cross-account, OIDC, estrutura da pipeline |
| 010 | [CDK Pipelines — Stages e ShellSteps](session-010-cdk-pipelines-stages-shellsteps.md) | Stages customizados, ShellSteps, self-mutation |
| 011 | [CustomResources e Aspects](session-011-cdk-custom-resources-aspects.md) | Custom Resources com Lambda, Aspects para validação global |
| 012 | [Context e feature flags](session-012-cdk-context-feature-flags.md) | cdk.json, context lookups, feature flags para produção 

### Módulo 3: ECS

| # | Guia | Tópico |
|---|------|--------|
| 013 | [Task Definitions](session-013-ecs-task-definitions-logging.md) | Containers, volumes, logging, resource limits |
| 014 | [Services e ALB](session-014-ecs-services-discovery-alb.md) | Services, service discovery, integração com ALB |
| 015 | [Fargate networking e IAM](session-015-ecs-fargate-networking-iam.md) | Networking, security groups, Task Role vs Execution Role |
| 016 | [Capacity Providers e autoscaling](session-016-ecs-capacity-providers-autoscaling.md) | Capacity Providers, Application Auto Scaling |
| 017 | [Deploy strategies](session-017-ecs-deploy-rolling-blue-green.md) | Rolling update, blue/green com CodeDeploy |
| 018 | [Observabilidade ECS](session-018-ecs-observabilidade-firelens-xray.md) | FireLens, Container Insights, X-Ray sidecar |

### Módulo 4: Lambda

| # | Guia | Tópico |
|---|------|--------|
| 019 | [Execution model e cold starts](session-019-lambda-execution-cold-starts.md) | Init phases, cold starts, provisioned concurrency |
| 020 | [Event source mappings](session-020-lambda-event-source-mappings.md) | SQS, Kinesis e DynamoDB Streams com filtering |
| 021 | [Extensions, Layers e Powertools](session-021-lambda-extensions-layers-powertools.md) | Lambda Extensions, Layers, AWS Lambda Powertools |
| 024 | [Observabilidade Lambda](session-024-lambda-observabilidade-xray-insights.md) | Structured logging, X-Ray, Lambda Insights |
| 025 | [Lambda@Edge vs CloudFront Functions](session-025-lambda-edge-cloudfront-functions.md) | Casos de uso, limites, quando escolher cada abordagem |

### Módulo 5: Orquestração com Step Functions

| # | Guia | Tópico |
|---|------|--------|
| 022 | [Standard vs Express e states básicos](session-022-stepfunctions-standard-express-states.md) | Task, Choice, Wait, Pass — diferenças de preço e semântica |
| 023 | [Parallel, Map e error handling](session-023-stepfunctions-parallel-map-error.md) | Paralelismo, iteração sobre arrays, retry/catch |

### Módulo 6: Load Balancers

| # | Guia | Tópico |
|---|------|--------|
| 026 | [ALB — listener rules avançadas](session-026-alb-listener-rules-weighted.md) | Listener rules complexas, weighted routing, fixed responses |
| 027 | [ALB — OIDC, mTLS e WAF](session-027-alb-oidc-mtls-waf.md) | Autenticação OIDC nativa, mutual TLS, integração com WAF |
| 028 | [NLB e GLB](session-028-nlb-glb-casos-uso.md) | NLB (TLS passthrough), GLB (inline inspection) |

### Módulo 7: DynamoDB

| # | Guia | Tópico |
|---|------|--------|
| 029 | [Access patterns first e PK/SK](session-029-dynamodb-access-patterns-pk-sk.md) | Modelagem orientada a access patterns, PK/SK genéricos |
| 030 | [Single-table design](session-030-dynamodb-single-table-adjacency.md) | Adjacency list, overloaded indexes, entidades em uma tabela |
| 031 | [GSIs e LSIs](session-031-dynamodb-gsi-lsi-hot-partitions.md) | Hot partitions, write amplification, custo de GSIs |
| 032 | [Streams e CDC](session-032-dynamodb-streams-lambda-cdc.md) | DynamoDB Streams com Lambda, Change Data Capture |
| 033 | [DAX](session-033-dax-arquitetura-casos-uso.md) | Arquitetura do DAX, casos de uso e quando NÃO usar |
| 034 | [Transações e operações condicionais](session-034-dynamodb-transacoes-condicionais.md) | TransactWriteItems, ConditionExpression, garantias e limites |
| 035 | [Global Tables](session-035-dynamodb-global-tables-consistencia.md) | Consistência eventual multi-região, conflict resolution |

### Módulo 8: Observabilidade

| # | Guia | Tópico |
|---|------|--------|
| 036 | [CloudWatch Custom Metrics (EMF)](session-036-cloudwatch-custom-metrics-emf.md) | Embedded Metrics Format, métricas custom sem custo de PutMetricData |
| 037 | [CloudWatch Logs Insights](session-037-cloudwatch-logs-insights-queries.md) | Sintaxe de queries, campos derivados, painéis |
| 038 | [Composite Alarms e anomaly detection](session-038-cloudwatch-composite-alarms-anomaly.md) | Alarmes compostos, detecção de anomalias, ações automatizadas |
| 039 | [X-Ray avançado](session-039-xray-groups-annotations-sampling.md) | Groups, annotations, sampling rules, integração cross-service |

### Módulo 9: FinOps

| # | Guia | Tópico |
|---|------|--------|
| 040 | [Savings Plans vs Reserved Instances](session-040-finops-savings-plans-reserved.md) | Diferenças, flexibilidade, quando usar cada modelo |
| 041 | [Spot Instances e Spot Fleet](session-041-finops-spot-instances-fleet.md) | Interruption handling, diversificação, Spot Fleet strategies |
| 042 | [Cost Explorer e Compute Optimizer](session-042-cost-explorer-anomaly-optimizer.md) | Cost Explorer, anomaly detection, rightsizing de instâncias |
| 043 | [Budgets e tagging strategy](session-043-budgets-acoes-automaticas-tagging.md) | Budgets com ações automáticas, tagging para chargeback |

### Módulo 10: Secrets e Parâmetros

| # | Guia | Tópico |
|---|------|--------|
| 044 | [Secrets Manager](session-044-secrets-manager-rotacao-rds.md) | Rotação automática, Lambda rotators, integração com RDS |
| 045 | [SSM Parameter Store](session-045-ssm-parameter-store-hierarquias.md) | Hierarquias, SecureString, custo vs Secrets Manager |

### Módulo 11: EKS

| # | Guia | Tópico |
|---|------|--------|
| 046 | [Cluster provisioning e VPC CNI](session-046-eks-cluster-provisioning-cni.md) | eksctl, networking VPC CNI, modos de autenticação |
| 047 | [Node groups e Fargate](session-047-eks-node-groups-fargate.md) | Managed node groups vs Fargate profiles, upgrades, node drain |
| 048 | [IRSA e Pod Identity](session-048-eks-irsa-pod-identity.md) | IAM para workloads sem credenciais estáticas — dois modelos comparados |
| 049 | [Add-ons gerenciados](session-049-eks-addons-vpc-cni-ebs.md) | VPC CNI (Prefix Delegation), CoreDNS, EBS CSI Driver |
| 050 | [Karpenter](session-050-eks-karpenter-node-provisioning.md) | Provisionamento dinâmico de nodes, NodePool, EC2NodeClass |

### Módulo 12: Bancos de dados gerenciados

| # | Guia | Tópico |
|---|------|--------|
| 051 | [RDS — HA e proxy](session-051-rds-multi-az-proxy-insights.md) | Multi-AZ, read replicas, RDS Proxy (pooling + IAM auth), Performance Insights |
| 052 | [Aurora — serverless e global](session-052-aurora-serverless-global-database.md) | Aurora serverless (ACUs, auto-pause), Global Database, diferenças vs RDS |

---
## Tecnologias cobertas

| Área | Serviços |
|------|----------|
| IaC | AWS CLI, CloudFormation, CDK v2 (Python) |
| Compute | ECS Fargate, Lambda, Lambda@Edge, CloudFront Functions |
| Orquestração | Step Functions (Standard + Express) |
| Containers | EKS, managed node groups, Fargate profiles, Karpenter |
| Networking | ALB, NLB, GLB, VPC CNI |
| Banco de dados | DynamoDB, DAX, RDS Multi-AZ, RDS Proxy, Aurora serverless, Aurora Global Database |
| Observabilidade | CloudWatch (EMF, Logs Insights, Composite Alarms, Anomaly Detection), X-Ray |
| Segurança | IRSA, Pod Identity, Secrets Manager, SSM Parameter Store, WAF, mTLS, OIDC |
| FinOps | Savings Plans, Reserved Instances, Spot Fleet, Cost Explorer, Budgets |
