# Sessão 050 — EKS: Karpenter — Provisionamento Dinâmico de Nodes e NodePools

**Duração estimada:** 60 min  
**Pré-requisito:** session-049 (EKS add-ons, VPC CNI, EBS CSI)

---

## Objetivos da sessão

- Entender a arquitetura do Karpenter e como ele difere do Cluster Autoscaler
- Instalar Karpenter via Helm com as políticas IAM corretas
- Criar NodePool e EC2NodeClass com restrições de instância, taint/toleration e disruption budgets
- Verificar que o Karpenter provisiona e consolida nodes em resposta a pods Pending/removidos
- Decidir quando usar Karpenter vs Cluster Autoscaler vs managed node groups

---

## 1. Karpenter vs Cluster Autoscaler — Comparação

### 1.1 Modelo mental

[FATO] O Cluster Autoscaler (CAS) opera no nível de **Auto Scaling Groups (ASGs)**: quando o scheduler Kubernetes não consegue colocar um pod, o CAS verifica quais ASGs poderiam absorver o pod e aumenta a `desiredCapacity` do grupo. O Karpenter opera no nível de **pods diretamente**: quando o scheduler não consegue alocar um pod, o Karpenter chama diretamente a EC2 API para criar a instância que melhor atende aos recursos solicitados.

```
┌─────────────────────────────────────────────────────────────────────┐
│ Cluster Autoscaler                                                  │
│                                                                     │
│  Pod Pending                                                        │
│       │                                                             │
│       ▼                                                             │
│  CAS verifica node groups pré-configurados                         │
│  (cada node group = 1 tipo de instância ou familia restrita)        │
│       │                                                             │
│       ▼                                                             │
│  Aumenta desiredCapacity do ASG mais adequado                       │
│       │                                                             │
│       ▼                                                             │
│  EC2 Auto Scaling cria instância (pode levar 2-5 min)              │
│       │                                                             │
│       ▼                                                             │
│  Node registra no cluster → pod é agendado                         │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│ Karpenter                                                           │
│                                                                     │
│  Pod Pending (watch via K8s API)                                    │
│       │                                                             │
│       ▼                                                             │
│  Karpenter lê requirements do pod (resources, nodeSelector,         │
│  affinity, tolerations, topologySpread)                             │
│       │                                                             │
│       ▼                                                             │
│  Seleciona instância mais econômica que atende os requirements      │
│  de todos os pods pendentes simultaneamente (bin-packing)           │
│       │                                                             │
│       ▼                                                             │
│  Chama EC2 Fleet API diretamente (RunInstances / CreateFleet)       │
│  Cria NodeClaim CRD para rastrear o estado                          │
│       │                                                             │
│       ▼                                                             │
│  Node registra → pod agendado (geralmente em < 60s)                 │
└─────────────────────────────────────────────────────────────────────┘
```

[FATO] Comparação estrutural:

```
╔══════════════════════════╦═══════════════════════════╦═══════════════════════════╗
║ Dimensão                 ║ Cluster Autoscaler (CAS)  ║ Karpenter                 ║
╠══════════════════════════╬═══════════════════════════╬═══════════════════════════╣
║ Unidade de escala        ║ Node Group (ASG)          ║ Instância EC2 individual  ║
║ Decisão de instância     ║ Pré-configurada no NG     ║ Runtime (melhor fit)      ║
║ Nº de node groups        ║ Muitos (1 por workload)   ║ Poucos (1-3 NodePools)    ║
║ Velocidade de scale-up   ║ 2-5 min (ASG trigger)     ║ < 60s (EC2 API direto)    ║
║ Consolidação (scale-down)║ 10 min de idle padrão     ║ Configurável (1m+)        ║
║ Versionamento K8s        ║ Acoplado (versão-específi)║ Desacoplado               ║
║ Spot diversidade         ║ Manual (1 NG p/ família)  ║ Automático (pool amplo)   ║
║ Bin-packing              ║ Limitado (por ASG)        ║ Global (todos os pods)    ║
║ AWS API                  ║ Auto Scaling API          ║ EC2 Fleet / RunInstances  ║
╚══════════════════════════╩═══════════════════════════╩═══════════════════════════╝
```

[CONSENSO] **Quando preferir Karpenter**: clusters com demanda variável/espigada, workloads heterogêneos, necessidade de Spot com alta diversidade de instâncias, ou quando o overhead de manter dezenas de node groups é excessivo.

[CONSENSO] **Quando preferir CAS ou node groups estáticos**: workloads estáveis e previsíveis, quando restrições organizacionais impedem IAM com poderes de RunInstances/TerminateInstances amplos, ou clusters com necessidade de conformidade usando configurações de nó muito específicas.

---

## 2. Arquitetura do Karpenter

### 2.1 Componentes

[FATO] O Karpenter roda como um **Deployment** com 2 réplicas (controller + webhook) em `kube-system`. Ele **não** é um managed EKS add-on — é instalado via Helm chart do OCI registry `public.ecr.aws/karpenter/karpenter`.

[FATO] CRDs criados pelo Karpenter:

```
karpenter.sh/v1:
  NodePool      — restrições de scheduling e políticas de disruption
  NodeClaim     — representa uma instância EC2 em provisionamento/ativa

karpenter.k8s.aws/v1:
  EC2NodeClass  — configuração AWS-específica (AMI, subnet, SG, role)

karpenter.sh/v1 (readonly):
  NodeOverlay   — sobreposição de configuração sobre EC2NodeClass existente
```

[FATO] O Karpenter deve rodar em um nó **não gerenciado por ele mesmo** — em um managed node group ou em Fargate. Se o único nó do cluster for provisionado pelo Karpenter e ele precisar ser removido (consolidação), o Karpenter controller ficaria sem onde rodar.

### 2.2 Fluxo de provisioning

```
1. Pod fica Pending (scheduler não encontra nó adequado)
2. Karpenter detecta o pod via watch na K8s API
3. Karpenter agrupa pods pendentes que podem ser co-localizados
4. Seleciona EC2NodeClass e NodePool adequados
5. Escolhe instância ótima (bin-packing + custo + disponibilidade)
6. Cria NodeClaim CRD (rastreia estado)
7. Chama EC2 Fleet API para criar a instância
8. Instância bootstrapping: nodeadm/userdata configura o kubelet
9. Node aparece no kubectl get nodes
10. Karpenter associa o NodeClaim ao Node
11. Scheduler agenda os pods no novo nó
```

### 2.3 IAM — Políticas do Karpenter

[FATO] O CloudFormation da instalação oficial cria 6 políticas IAM separadas (v1.13):

```
KarpenterControllerNodeLifecyclePolicy     → RunInstances, TerminateInstances,
                                             CreateFleet, CreateLaunchTemplate,
                                             DeleteLaunchTemplate, ...
KarpenterControllerIAMIntegrationPolicy    → iam:PassRole (para KarpenterNodeRole),
                                             iam:AddRoleToInstanceProfile,
                                             iam:CreateInstanceProfile, ...
KarpenterControllerEKSIntegrationPolicy    → eks:DescribeCluster
KarpenterControllerInterruptionPolicy      → sqs:ReceiveMessage, sqs:DeleteMessage,
                                             sqs:GetQueueUrl,
                                             events:CreateEventBus, ...
KarpenterControllerResourceDiscoveryPolicy → ec2:Describe* (instâncias, AZs, subnets,
                                             SGs), pricing:GetProducts
KarpenterControllerZonalShiftPolicy        → arc-zonal-shift:GetManagedResource
```

[FATO] O `KarpenterNodeRole-<cluster>` é o IAM Role atribuído aos nós EC2 criados pelo Karpenter. Deve ter: `AmazonEKSWorkerNodePolicy`, `AmazonEKS_CNI_Policy`, `AmazonEC2ContainerRegistryReadOnly`, `AmazonSSMManagedInstanceCore`.

[FATO] **Risco de segurança de tags**: o Karpenter usa 3 tags para associar instâncias EC2 a NodeClaims:
- `karpenter.sh/managed-by: <cluster-name>`
- `karpenter.sh/nodepool: <nodepool-name>`
- `kubernetes.io/cluster/<cluster-name>: owned`

Qualquer usuário com `ec2:CreateTags`/`ec2:DeleteTags` nessas tags em instâncias `i-*` pode manipular o Karpenter. A recomendação é usar IAM policies baseadas em tags para restringir `CreateTags`/`DeleteTags` somente ao role do Karpenter.

---

## 3. NodePool e EC2NodeClass — Anatomia Completa

### 3.1 NodePool

[FATO] O NodePool define as restrições sobre os nós que o Karpenter pode criar. Cada pod Pending é comparado com os NodePools disponíveis e agendado no NodePool que melhor atende.

```yaml
# NodePool anotado com todos os campos relevantes
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: general-compute
spec:
  # ── Template: configuração dos nós que serão criados ──────────────
  template:
    metadata:
      labels:
        team: platform        # labels propagadas para o Node K8s
      annotations:
        example.com/owner: platform-team
    spec:
      # Referência ao EC2NodeClass (configuração AWS-específica)
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default

      # Taints no nó — pods precisam tolerar para serem agendados aqui
      taints: []

      # startupTaints: aplicados ao nó, mas pods NÃO precisam tolerar.
      # Usados para aguardar inicialização (ex: Cilium CNI agent).
      # Um DaemonSet ou controller externo deve remover o taint.
      startupTaints: []

      # Expiração do nó (TTL): após 720h, o nó é drenado e terminado.
      # Útil para forçar rotação e aplicar patches de OS/K8s.
      # 'Never' desabilita a expiração.
      expireAfter: 720h

      # Tempo máximo de drain antes de forçar terminação
      terminationGracePeriod: 48h

      # Requirements: constraints de scheduling (interseção com pod spec)
      # Operadores: In, NotIn, Exists, DoesNotExist, Gt, Lt, Gte, Lte
      requirements:
        # Arquitetura
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]

        # OS
        - key: kubernetes.io/os
          operator: In
          values: ["linux"]

        # Tipo de capacidade
        # Prioridade automática: reserved > spot > on-demand
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]

        # Categorias de instância (c=compute, m=general, r=memory)
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["c", "m", "r"]
          # minValues: exige pelo menos N categorias distintas no pool
          # (evita overfitting em uma única família para Spot)
          minValues: 2

        # Geração mínima (evita instâncias antigas)
        - key: karpenter.k8s.aws/instance-generation
          operator: Gte
          values: ["3"]

        # Excluir instâncias bare-metal (geralmente não necessárias)
        - key: karpenter.k8s.aws/instance-hypervisor
          operator: In
          values: ["nitro"]

  # ── Disruption: controle de consolidação e rotação ────────────────
  disruption:
    # WhenEmptyOrUnderutilized: consolida nós vazios E subutilizados
    # WhenEmpty: consolida apenas nós sem workload pods
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 1m    # aguarda 1 min de inatividade antes de consolidar

    # Budgets: limita quantos nós podem ser interrompidos simultaneamente
    budgets:
      - nodes: "10%"        # máximo 10% dos nós disruptados de uma vez
      # Durante horário comercial (seg-sex 9h-17h): sem disruption
      - schedule: "0 9 * * mon-fri"
        duration: 8h
        nodes: "0"

  # ── Limits: teto de recursos que este NodePool pode consumir ──────
  limits:
    cpu: "1000"       # 1000 vCPUs totais
    memory: 1000Gi    # 1 TiB de memória total
    # nodes: 50       # opcional: máximo de nós

  # ── Weight: prioridade quando múltiplos NodePools são candidatos ──
  weight: 10
```

### 3.2 EC2NodeClass

[FATO] A EC2NodeClass contém toda a configuração AWS-específica. Múltiplos NodePools podem referenciar a mesma EC2NodeClass.

```yaml
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  # IAM role para os nós EC2 (deve existir com as políticas de worker node)
  role: "KarpenterNodeRole-checkout-prod"

  # AMI: 'alias' permite usar a AMI EKS otimizada mais recente
  # Formatos: al2023@latest, al2023@v20240101, al2@latest, bottlerocket@latest
  amiSelectorTerms:
    - alias: "al2023@latest"

  # Subnets: Karpenter usa tags para descobrir subnets disponíveis
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: "checkout-prod"
    # Alternativa: por ID de subnet
    # - id: subnet-0abc123

  # Security Groups: mesma lógica de tags
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: "checkout-prod"

  # Kubelet: configuração do kubelet nos nós (movido de NodePool para EC2NodeClass)
  kubelet:
    maxPods: 110      # aumentar se usar Prefix Delegation (ex: 737 para m5.xlarge)
    systemReserved:
      cpu: "100m"
      memory: "100Mi"
      ephemeral-storage: "1Gi"
    kubeReserved:
      cpu: "100m"
      memory: "200Mi"
      ephemeral-storage: "3Gi"
    evictionHard:
      memory.available: "5%"
      nodefs.available: "10%"

  # Block device mapping: tamanho e criptografia do volume root
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 50Gi
        volumeType: gp3
        encrypted: true
        iops: 3000
        throughput: 125

  # Tags adicionais a todos os nós criados
  tags:
    Environment: production
    ManagedBy: karpenter

  # userData: script adicional executado no bootstrap (RARE — preferir AMI customizada)
  # userData: |
  #   #!/bin/bash
  #   echo "custom init" >> /var/log/init.log
```

---

## 4. CDK Python — Instalação do Karpenter

```python
"""
CDK Stack para instalar o Karpenter em um cluster EKS existente.
Usa Pod Identity (preferido em v1.13) para o controller role.
"""
from aws_cdk import (
    Stack, CfnOutput,
    aws_eks as eks,
    aws_iam as iam,
    aws_sqs as sqs,
    aws_ec2 as ec2,
)
from constructs import Construct


class KarpenterStack(Stack):
    def __init__(self, scope: Construct, construct_id: str,
                 cluster: eks.Cluster, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        CLUSTER_NAME = cluster.cluster_name

        # ──────────────────────────────────────────────────────────────
        # 1. IAM Role para os NODOS criados pelo Karpenter
        #    (não confundir com o role do Karpenter controller)
        # ──────────────────────────────────────────────────────────────
        node_role = iam.Role(self, "KarpenterNodeRole",
            role_name=f"KarpenterNodeRole-{CLUSTER_NAME}",
            description="IAM role para EC2 nodes criados pelo Karpenter",
            assumed_by=iam.ServicePrincipal("ec2.amazonaws.com"),
            managed_policies=[
                iam.ManagedPolicy.from_aws_managed_policy_name("AmazonEKSWorkerNodePolicy"),
                iam.ManagedPolicy.from_aws_managed_policy_name("AmazonEKS_CNI_Policy"),
                iam.ManagedPolicy.from_aws_managed_policy_name("AmazonEC2ContainerRegistryReadOnly"),
                iam.ManagedPolicy.from_aws_managed_policy_name("AmazonSSMManagedInstanceCore"),
            ],
        )

        # Instance profile (obrigatório para EC2 usar o role)
        node_instance_profile = iam.CfnInstanceProfile(self, "KarpenterNodeInstanceProfile",
            instance_profile_name=f"KarpenterNodeRole-{CLUSTER_NAME}",
            roles=[node_role.role_name],
        )

        # Adicionar o node role ao aws-auth (EKS Access Entries)
        cluster.grant_access("KarpenterNodeAccess",
            principal=node_role.role_arn,
            access_policies=[
                eks.AccessPolicy.from_access_policy_name(
                    "AmazonEKSWorkerNodePolicy",
                    access_scope=eks.AccessScope(type=eks.AccessScopeType.CLUSTER),
                ),
            ],
        )

        # ──────────────────────────────────────────────────────────────
        # 2. SQS Queue para interruption handling (Spot + maintenance)
        # ──────────────────────────────────────────────────────────────
        interruption_queue = sqs.Queue(self, "KarpenterInterruptionQueue",
            queue_name=CLUSTER_NAME,    # nome deve ser = cluster name
            retention_period=None,
        )

        # Permite que EC2 e SQS publiquem eventos de interrupção na fila
        interruption_queue.add_to_resource_policy(iam.PolicyStatement(
            principals=[
                iam.ServicePrincipal("sqs.amazonaws.com"),
                iam.ServicePrincipal("events.amazonaws.com"),
            ],
            actions=["sqs:SendMessage"],
            resources=[interruption_queue.queue_arn],
        ))

        # ──────────────────────────────────────────────────────────────
        # 3. IAM Role do Karpenter Controller (via Pod Identity)
        # ──────────────────────────────────────────────────────────────
        controller_role = iam.Role(self, "KarpenterControllerRole",
            role_name=f"{CLUSTER_NAME}-karpenter",
            description="Karpenter controller role — chama EC2 API para criar/terminar nós",
            assumed_by=iam.ServicePrincipal("pods.eks.amazonaws.com"),
        )
        controller_role.assume_role_policy.add_statements(
            iam.PolicyStatement(
                effect=iam.Effect.ALLOW,
                principals=[iam.ServicePrincipal("pods.eks.amazonaws.com")],
                actions=["sts:AssumeRole", "sts:TagSession"],
            )
        )

        # Política de lifecycle de nós (RunInstances, TerminateInstances, etc.)
        controller_role.add_to_policy(iam.PolicyStatement(
            sid="NodeLifecycle",
            actions=[
                "ec2:RunInstances",
                "ec2:CreateFleet",
                "ec2:CreateLaunchTemplate",
                "ec2:DeleteLaunchTemplate",
                "ec2:TerminateInstances",
                "ec2:CreateTags",
                "ec2:DeleteTags",
            ],
            resources=["*"],
            conditions={
                "StringEquals": {
                    f"aws:RequestedRegion": self.region,
                }
            },
        ))

        # Política de descoberta de recursos
        controller_role.add_to_policy(iam.PolicyStatement(
            sid="ResourceDiscovery",
            actions=[
                "ec2:DescribeAvailabilityZones",
                "ec2:DescribeImages",
                "ec2:DescribeInstances",
                "ec2:DescribeInstanceTypes",
                "ec2:DescribeInstanceTypeOfferings",
                "ec2:DescribeSecurityGroups",
                "ec2:DescribeSpotPriceHistory",
                "ec2:DescribeSubnets",
                "ssm:GetParameter",
                "pricing:GetProducts",
            ],
            resources=["*"],
        ))

        # Passagem de role para instâncias (IAMIntegration)
        controller_role.add_to_policy(iam.PolicyStatement(
            sid="IAMIntegration",
            actions=["iam:PassRole"],
            resources=[node_role.role_arn],
            conditions={
                "StringEquals": {"iam:PassedToService": "ec2.amazonaws.com"},
            },
        ))

        controller_role.add_to_policy(iam.PolicyStatement(
            sid="IAMInstanceProfile",
            actions=[
                "iam:AddRoleToInstanceProfile",
                "iam:CreateInstanceProfile",
                "iam:DeleteInstanceProfile",
                "iam:GetInstanceProfile",
                "iam:RemoveRoleFromInstanceProfile",
                "iam:TagInstanceProfile",
                "iam:UntagInstanceProfile",
            ],
            resources=["*"],
        ))

        # Acesso à fila de interrupção
        interruption_queue.grant_consume_messages(controller_role)
        controller_role.add_to_policy(iam.PolicyStatement(
            sid="InterruptionQueue",
            actions=["sqs:GetQueueUrl", "sqs:GetQueueAttributes"],
            resources=[interruption_queue.queue_arn],
        ))

        # EKS DescribeCluster
        controller_role.add_to_policy(iam.PolicyStatement(
            sid="EKSIntegration",
            actions=["eks:DescribeCluster"],
            resources=[cluster.cluster_arn],
        ))

        # ──────────────────────────────────────────────────────────────
        # 4. Pod Identity Association para o controller
        # ──────────────────────────────────────────────────────────────
        eks.CfnPodIdentityAssociation(self, "KarpenterPodIdentity",
            cluster_name=CLUSTER_NAME,
            namespace="kube-system",
            service_account="karpenter",
            role_arn=controller_role.role_arn,
        )

        # ──────────────────────────────────────────────────────────────
        # 5. Tags nas subnets e security groups para descoberta
        #    (Karpenter usa tags para descobrir resources via EC2 API)
        # ──────────────────────────────────────────────────────────────
        # NOTA: no CDK, adicionar tags via vpc.select_subnets() + tags
        # Na prática, mais fácil via eksctl ou CLI:
        # aws ec2 create-tags --resources <subnet-ids> \
        #   --tags Key=karpenter.sh/discovery,Value=<cluster-name>

        # ──────────────────────────────────────────────────────────────
        # 6. Helm chart do Karpenter
        # ──────────────────────────────────────────────────────────────
        karpenter_chart = cluster.add_helm_chart("Karpenter",
            chart="karpenter",
            repository="oci://public.ecr.aws/karpenter/karpenter",
            version="1.13.0",
            namespace="kube-system",
            create_namespace=False,
            values={
                "settings": {
                    "clusterName": CLUSTER_NAME,
                    "interruptionQueue": CLUSTER_NAME,
                    "enableZonalShift": True,
                },
                "controller": {
                    "resources": {
                        "requests": {"cpu": "1", "memory": "1Gi"},
                        "limits":   {"cpu": "1", "memory": "1Gi"},
                    },
                },
                # dnsPolicy: Default se CoreDNS roda em nós Karpenter
                # "dnsPolicy": "ClusterFirst",  # padrão
            },
            wait=True,
        )

        CfnOutput(self, "KarpenterControllerRoleArn",
            value=controller_role.role_arn)
        CfnOutput(self, "KarpenterInterruptionQueueUrl",
            value=interruption_queue.queue_url)
```

---

## 5. Python — Geração de NodePools por perfil de workload

```python
"""
Gera manifestos NodePool + EC2NodeClass para diferentes perfis.
Útil para aplicar via kubectl apply ou via cluster.add_manifest() no CDK.
"""
import yaml
from typing import Literal


def render_ec2_node_class(
    name: str,
    cluster_name: str,
    root_volume_size_gi: int = 50,
    max_pods: int = 110,
    custom_kubelet: dict | None = None,
) -> dict:
    """EC2NodeClass compartilhada por múltiplos NodePools."""
    kubelet_config = {
        "maxPods": max_pods,
        "systemReserved": {"cpu": "100m", "memory": "100Mi"},
        "kubeReserved":   {"cpu": "100m", "memory": "200Mi"},
        "evictionHard": {"memory.available": "5%", "nodefs.available": "10%"},
    }
    if custom_kubelet:
        kubelet_config.update(custom_kubelet)

    return {
        "apiVersion": "karpenter.k8s.aws/v1",
        "kind": "EC2NodeClass",
        "metadata": {"name": name},
        "spec": {
            "role": f"KarpenterNodeRole-{cluster_name}",
            "amiSelectorTerms": [{"alias": "al2023@latest"}],
            "subnetSelectorTerms": [{"tags": {"karpenter.sh/discovery": cluster_name}}],
            "securityGroupSelectorTerms": [{"tags": {"karpenter.sh/discovery": cluster_name}}],
            "kubelet": kubelet_config,
            "blockDeviceMappings": [{
                "deviceName": "/dev/xvda",
                "ebs": {
                    "volumeSize": f"{root_volume_size_gi}Gi",
                    "volumeType": "gp3",
                    "encrypted": True,
                },
            }],
            "tags": {"ManagedBy": "karpenter", "Cluster": cluster_name},
        },
    }


def render_nodepool(
    name: str,
    node_class_name: str,
    capacity_types: list[str],
    instance_categories: list[str],
    min_instance_categories: int = 2,
    taints: list[dict] | None = None,
    labels: dict | None = None,
    expire_after: str = "720h",
    consolidation_policy: Literal["WhenEmpty", "WhenEmptyOrUnderutilized"] = "WhenEmptyOrUnderutilized",
    consolidate_after: str = "1m",
    cpu_limit: str = "200",
    weight: int = 10,
) -> dict:
    """NodePool genérico com configurações parametrizadas."""
    requirements = [
        {"key": "kubernetes.io/arch",   "operator": "In", "values": ["amd64"]},
        {"key": "kubernetes.io/os",     "operator": "In", "values": ["linux"]},
        {"key": "karpenter.sh/capacity-type", "operator": "In", "values": capacity_types},
        {
            "key": "karpenter.k8s.aws/instance-category",
            "operator": "In",
            "values": instance_categories,
            "minValues": min_instance_categories,
        },
        {"key": "karpenter.k8s.aws/instance-generation", "operator": "Gte", "values": ["3"]},
        {"key": "karpenter.k8s.aws/instance-hypervisor", "operator": "In", "values": ["nitro"]},
    ]

    spec_node = {
        "nodeClassRef": {"group": "karpenter.k8s.aws", "kind": "EC2NodeClass", "name": node_class_name},
        "expireAfter": expire_after,
        "requirements": requirements,
    }
    if taints:
        spec_node["taints"] = taints

    template: dict = {"spec": spec_node}
    if labels:
        template["metadata"] = {"labels": labels}

    return {
        "apiVersion": "karpenter.sh/v1",
        "kind": "NodePool",
        "metadata": {"name": name},
        "spec": {
            "template": template,
            "disruption": {
                "consolidationPolicy": consolidation_policy,
                "consolidateAfter": consolidate_after,
                "budgets": [
                    {"nodes": "10%"},
                    {"schedule": "0 9 * * mon-fri", "duration": "8h", "nodes": "0"},
                ],
            },
            "limits": {"cpu": cpu_limit},
            "weight": weight,
        },
    }


def generate_cluster_nodepools(cluster_name: str) -> list[dict]:
    """
    Gera 3 NodePools + 1 EC2NodeClass para um cluster de produção típico:
    1. general-od  — on-demand, para workloads críticos
    2. general-spot — spot, para workloads tolerantes a interrupção
    3. gpu         — on-demand p3/g4, para ML workloads (com taint)
    """
    manifests = []

    # EC2NodeClass compartilhada
    manifests.append(render_ec2_node_class("default", cluster_name))
    # EC2NodeClass para GPU (volume maior)
    manifests.append(render_ec2_node_class("gpu", cluster_name, root_volume_size_gi=100))

    # NodePool 1: On-demand para workloads críticos
    manifests.append(render_nodepool(
        name="general-od",
        node_class_name="default",
        capacity_types=["on-demand"],
        instance_categories=["c", "m", "r"],
        min_instance_categories=2,
        expire_after="720h",
        cpu_limit="500",
        weight=10,
    ))

    # NodePool 2: Spot para workloads tolerantes (batch, workers)
    manifests.append(render_nodepool(
        name="general-spot",
        node_class_name="default",
        capacity_types=["spot"],
        instance_categories=["c", "m", "r"],
        min_instance_categories=3,   # maior diversidade = menor risco de interrupção
        expire_after="168h",         # 7 dias (mais curto para spot)
        consolidation_policy="WhenEmptyOrUnderutilized",
        consolidate_after="30s",     # consolidar mais rápido para Spot
        cpu_limit="1000",
        labels={"karpenter.sh/capacity-type-preference": "spot"},
        weight=5,                    # peso menor = segunda opção
    ))

    # NodePool 3: GPU com taint para isolar workloads ML
    manifests.append({
        "apiVersion": "karpenter.sh/v1",
        "kind": "NodePool",
        "metadata": {"name": "gpu"},
        "spec": {
            "template": {
                "spec": {
                    "nodeClassRef": {"group": "karpenter.k8s.aws", "kind": "EC2NodeClass", "name": "gpu"},
                    "taints": [{"key": "nvidia.com/gpu", "value": "true", "effect": "NoSchedule"}],
                    "expireAfter": "Never",   # GPU nodes não expiram (caros para substituir)
                    "requirements": [
                        {"key": "kubernetes.io/os",  "operator": "In", "values": ["linux"]},
                        {"key": "kubernetes.io/arch", "operator": "In", "values": ["amd64"]},
                        {"key": "karpenter.sh/capacity-type", "operator": "In", "values": ["on-demand"]},
                        {
                            "key": "node.kubernetes.io/instance-type",
                            "operator": "In",
                            "values": ["p3.2xlarge", "p3.8xlarge", "g4dn.xlarge", "g4dn.2xlarge"],
                        },
                    ],
                },
            },
            "disruption": {
                "consolidationPolicy": "WhenEmpty",
                "consolidateAfter": "5m",
            },
            "limits": {"cpu": "128"},
            "weight": 20,   # maior peso = prioridade para pods GPU
        },
    })

    return manifests


if __name__ == "__main__":
    manifests = generate_cluster_nodepools("checkout-prod")
    for m in manifests:
        print("---")
        print(yaml.dump(m, default_flow_style=False))
```

---

## 6. CLI — Instalação e Operação do Karpenter

```bash
# ═══════════════════════════════════════════════════════════════
# Setup inicial
# ═══════════════════════════════════════════════════════════════

export CLUSTER_NAME="checkout-prod"
export KARPENTER_VERSION="1.13.0"
export K8S_VERSION="1.36"
export AWS_REGION="us-east-1"
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export AWS_PARTITION="aws"
export KARPENTER_NAMESPACE="kube-system"

# Obter versão da AMI EKS otimizada (para EC2NodeClass alias)
export ALIAS_VERSION=$(aws ssm get-parameter \
  --name "/aws/service/eks/optimized-ami/${K8S_VERSION}/amazon-linux-2023/x86_64/standard/recommended/image_id" \
  --query Parameter.Value \
  | xargs aws ec2 describe-images --query 'Images[0].Name' --image-ids \
  | sed -r 's/^.*(v[[:digit:]]+).*$/\1/')

echo "AMI alias version: al2023@${ALIAS_VERSION}"

# Criar service-linked role para Spot (necessário se nunca usado na conta)
aws iam create-service-linked-role --aws-service-name spot.amazonaws.com || true

# ─────────────────────────────────────────────────────────────
# Taggear subnets e security groups para descoberta do Karpenter
# ─────────────────────────────────────────────────────────────

# Obter subnets privadas do cluster
SUBNET_IDS=$(aws eks describe-cluster \
  --name "$CLUSTER_NAME" \
  --query 'cluster.resourcesVpcConfig.subnetIds[]' \
  --output text)

# Taggear subnets para descoberta
aws ec2 create-tags \
  --resources $SUBNET_IDS \
  --tags Key=karpenter.sh/discovery,Value="$CLUSTER_NAME"

# Taggear security group do cluster
CLUSTER_SG=$(aws eks describe-cluster \
  --name "$CLUSTER_NAME" \
  --query 'cluster.resourcesVpcConfig.clusterSecurityGroupId' \
  --output text)

aws ec2 create-tags \
  --resources "$CLUSTER_SG" \
  --tags Key=karpenter.sh/discovery,Value="$CLUSTER_NAME"

# ─────────────────────────────────────────────────────────────
# Instalar Karpenter via Helm (OCI registry)
# ─────────────────────────────────────────────────────────────

# Logout primeiro para pull anônimo do ECR público
helm registry logout public.ecr.aws 2>/dev/null || true

helm upgrade --install karpenter \
  oci://public.ecr.aws/karpenter/karpenter \
  --version "${KARPENTER_VERSION}" \
  --namespace "${KARPENTER_NAMESPACE}" \
  --set "settings.clusterName=${CLUSTER_NAME}" \
  --set "settings.interruptionQueue=${CLUSTER_NAME}" \
  --set "settings.enableZonalShift=true" \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=1Gi \
  --set controller.resources.limits.cpu=1 \
  --set controller.resources.limits.memory=1Gi \
  --wait

# Verificar instalação
kubectl get pods -n kube-system -l app.kubernetes.io/name=karpenter
kubectl get crd | grep karpenter

# ─────────────────────────────────────────────────────────────
# Criar EC2NodeClass e NodePool
# ─────────────────────────────────────────────────────────────

cat <<EOF | envsubst | kubectl apply -f -
---
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  role: "KarpenterNodeRole-${CLUSTER_NAME}"
  amiSelectorTerms:
    - alias: "al2023@${ALIAS_VERSION}"
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: "${CLUSTER_NAME}"
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: "${CLUSTER_NAME}"
  kubelet:
    maxPods: 110
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 50Gi
        volumeType: gp3
        encrypted: true
  tags:
    ManagedBy: karpenter
---
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
      expireAfter: 720h
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: kubernetes.io/os
          operator: In
          values: ["linux"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand"]
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["c", "m", "r"]
        - key: karpenter.k8s.aws/instance-generation
          operator: Gte
          values: ["3"]
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 1m
  limits:
    cpu: "1000"
EOF

# Verificar status do NodePool
kubectl get nodepool
kubectl describe nodepool default
# Verificar: status.conditions.type=Ready deve ser True

# ═══════════════════════════════════════════════════════════════
# Testar scale-up e scale-down
# ═══════════════════════════════════════════════════════════════

# Deploy de teste (pause containers, 1 CPU cada)
kubectl apply -f - << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inflate
spec:
  replicas: 0
  selector:
    matchLabels:
      app: inflate
  template:
    metadata:
      labels:
        app: inflate
    spec:
      terminationGracePeriodSeconds: 0
      containers:
      - name: inflate
        image: public.ecr.aws/eks-distro/kubernetes/pause:3.7
        resources:
          requests:
            cpu: "1"
EOF

# Scale up: 5 pods = 5 vCPUs → Karpenter provisiona um nó
kubectl scale deployment inflate --replicas 5

# Observar logs do Karpenter em tempo real
kubectl logs -f -n kube-system \
  -l app.kubernetes.io/name=karpenter \
  -c controller \
  --since=1m | grep -E "provisioned|launched|registered|scheduled"

# Verificar NodeClaim criado
kubectl get nodeclaims
kubectl describe nodeclaim <name>   # mostra qual instance type foi escolhido

# Verificar novo nó
kubectl get nodes -L karpenter.sh/nodepool,node.kubernetes.io/instance-type

# Scale down: Karpenter consolida e termina o nó
kubectl scale deployment inflate --replicas 0

# Após ~1 min, o nó deve ser terminado
kubectl get nodes --watch

# ─────────────────────────────────────────────────────────────
# Operações de manutenção
# ─────────────────────────────────────────────────────────────

# Listar nós por NodePool
kubectl get nodes -l karpenter.sh/nodepool=default

# Status de recursos consumidos pelo NodePool
kubectl get nodepool default -o jsonpath='{.status.resources}' | python3 -m json.tool

# Forçar consolidação imediata de um NodePool (drift)
kubectl annotate nodepool default \
  karpenter.sh/disruption-reason="manual-consolidation" \
  --overwrite

# Proteger um pod específico de disruption
kubectl annotate pod <pod-name> karpenter.sh/do-not-disrupt="true"

# Deletar um nó Karpenter de forma graciosa (drain + terminate EC2)
kubectl delete node <node-name>
# Karpenter tem um finalizer que garante drain antes de terminar a instância

# Ver métricas do Karpenter (se Prometheus disponível)
kubectl port-forward -n kube-system \
  deployment/karpenter 8080:8080

# Acessar em http://localhost:8080/metrics
# Métricas relevantes:
# karpenter_provisioner_scheduling_duration_seconds
# karpenter_nodes_total_daemon_requests
# karpenter_nodes_total_pod_requests
# karpenter_disruption_consolidation_timeouts_total
```

---

## 7. Armadilhas (Pitfalls)

[FATO] **Karpenter não deve gerenciar os próprios nós onde roda**: se o único managed node group do cluster for removido e o Karpenter controller tentar consolidar o nó onde ele próprio roda, haverá deadlock. Manter pelo menos 2 nós no managed node group do sistema (ou usar Fargate para `kube-system`).

[FATO] **NodePools sobrepostos sem `weight` definido causam comportamento aleatório**: se dois NodePools podem agendar o mesmo pod e não têm peso diferente, o Karpenter escolhe aleatoriamente. Usar `spec.weight` ou `taints`/`requirements` para torná-los mutuamente exclusivos.

[FATO] **Spot sem diversidade de instâncias causa `InsufficientInstanceCapacity`**: fixar apenas 1 ou 2 tipos de instância Spot é arriscado. O Karpenter usa `price-capacity-optimized` — deixar pelo menos 10 tipos de instância elegíveis. O `minValues` em `instance-family` força diversidade mínima.

[FATO] **pods sem `requests` definidos causam bin-packing incorreto**: o Karpenter dimensiona os nós com base em `requests` dos pods, não em `limits`. Pods sem `requests` são tratados como se não consumissem recursos, levando a nós subdimensionados e OOM kills. Usar `LimitRange` para definir defaults por namespace.

[FATO] **`expireAfter: Never` em NodePools de workloads gerais acumula CVEs**: nós de longa duração acumulam vulnerabilidades. O padrão de 720h (30 dias) garante rotação periódica com patches do sistema operacional.

[FATO] **Karpenter e Node Termination Handler (NTH) não devem coexistir**: o NTH e o mecanismo de interruption handling do Karpenter podem conflitar ao lidar com eventos Spot. Se o Karpenter está configurado com `interruptionQueue`, desinstalar o NTH.

[FATO] **DNS policy do controller**: o Karpenter usa `ClusterFirst` por padrão, o que cria dependência circular se o CoreDNS roda em nós gerenciados pelo Karpenter. Solução: manter CoreDNS em node group fixo, OU usar `dnsPolicy: Default` no Karpenter.

---

## Exercício de Reflexão

Um cluster EKS com 3 times tem os seguintes workloads:

- **API Team**: serviço crítico `checkout-api`, 20 réplicas com `requests: cpu=500m, memory=512Mi`. Não pode ser interrompido durante horário comercial (seg-sex 8h–20h). Deve rodar em instâncias `c` ou `m` generation 5+.
- **ML Team**: jobs de treinamento que rodam à noite, toleram interrupção (checkpoints a cada 5 min), precisam de GPUs `p3.2xlarge` ou `g4dn.xlarge`, e devem ter custo minimizado.
- **Platform Team**: DaemonSets e ferramentas de observabilidade que devem rodar em **todos** os nós.

**Responda:**

1. Quantos NodePools você criaria e qual a justificativa para cada um? Descreva os `requirements`, `taints`, `weight` e `disruption` de cada um.

2. O ML Team quer usar Spot para seus jobs de GPU. Quais riscos específicos existem com Spot para GPUs e como o Karpenter mitigaria esses riscos com a fila de interrupção SQS?

3. Para o `checkout-api`, como você garantiria que o Karpenter **não consolide** os nós durante horário comercial? (Descreva usando o campo correto da spec do NodePool).

4. O time de plataforma quer que suas ferramentas de observabilidade (DaemonSet) rodem em **todos** os nós criados pelo Karpenter, incluindo GPU nodes. Por que DaemonSets **não precisam** de tolerations para taints do tipo `NoSchedule` criados pelo Karpenter? (Dica: `startupTaints` vs `taints`).

5. Um arquiteto propõe migrar de Karpenter para EKS Auto Mode. Quais são as diferenças fundamentais entre EKS Auto Mode e Karpenter para esse cenário com múltiplos perfis de workload?

---

## Referências

- [FATO] [Getting Started with Karpenter (v1.13)](https://karpenter.sh/docs/getting-started/getting-started-with-karpenter/) — karpenter.sh
- [FATO] [NodePools — Karpenter v1.13](https://karpenter.sh/docs/concepts/nodepools/) — karpenter.sh
- [FATO] [NodeClasses — Karpenter v1.13](https://karpenter.sh/docs/concepts/nodeclasses/) — karpenter.sh
- [FATO] [Disruption — Karpenter v1.13](https://karpenter.sh/docs/concepts/disruption/) — karpenter.sh
- [FATO] [Karpenter Best Practices — EKS Best Practices](https://aws.github.io/aws-eks-best-practices/karpenter/) — aws.github.io
- [FATO] [Scale cluster compute with Karpenter and Cluster Autoscaler — Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/autoscaling.html) — docs.aws.amazon.com
