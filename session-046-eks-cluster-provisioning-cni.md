# Sessão 046 — EKS: Cluster Provisioning com eksctl e VPC CNI Networking

**Duração estimada:** 60 min  
**Pré-requisito:** session-015 (ECS Fargate networking e IAM)

---

## Objetivos da sessão

- Escrever um eksctl ClusterConfig YAML completo (não flags inline) e executar `eksctl create cluster -f`
- Configurar o kubeconfig com `aws eks update-kubeconfig` e entender o mecanismo de autenticação
- Entender o modelo de IP allocation do VPC CNI em Secondary IP Mode (ENIs, slots, warm pool)
- Calcular pod density por instance type usando a fórmula oficial
- Entender Prefix Delegation Mode como alternativa ao Secondary IP mode
- Verificar cluster com `kubectl get nodes`, `kubectl get pods -A`

---

## 1. Arquitetura de um Cluster EKS

[FATO] Um cluster EKS consiste em:

```
┌─────────────────────────────────────────────────────────────────┐
│                    AWS Account / VPC                            │
│                                                                 │
│  ┌─────────────────────────────────────┐                        │
│  │    EKS Control Plane (AWS managed)  │                        │
│  │    kube-apiserver                   │                        │
│  │    etcd                             │                        │
│  │    kube-scheduler                   │                        │
│  │    kube-controller-manager          │                        │
│  └────────────────┬────────────────────┘                        │
│                   │ Kubernetes API                              │
│  ┌────────────────▼────────────────────────────────────────┐   │
│  │                Data Plane (Customer-managed)             │   │
│  │                                                         │   │
│  │  ┌──────────────────┐  ┌──────────────────┐            │   │
│  │  │ Managed Node     │  │ Self-managed Node │            │   │
│  │  │ Group (EC2)      │  │ Group (EC2)       │            │   │
│  │  │                  │  │                  │            │   │
│  │  │ kubelet          │  │ kubelet          │            │   │
│  │  │ kube-proxy       │  │ kube-proxy       │            │   │
│  │  │ VPC CNI (aws-node│  │ VPC CNI          │            │   │
│  │  └──────────────────┘  └──────────────────┘            │   │
│  │                                                         │   │
│  │  ┌──────────────────────────────────┐                  │   │
│  │  │  Fargate Profile                 │                  │   │
│  │  │  (serverless — 1 Pod per node)   │                  │   │
│  │  └──────────────────────────────────┘                  │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

[FATO] Versões Kubernetes suportadas pelo EKS atualmente (fonte: docs.aws.amazon.com/eks, junho 2026):

```
╔═══════════════╦═════════════════════════╗
║ K8s version   ║ VPC CNI add-on version  ║
╠═══════════════╬═════════════════════════╣
║ 1.36          ║ v1.21.1-eksbuild.8      ║
║ 1.35          ║ v1.21.1-eksbuild.8      ║
║ 1.34          ║ v1.21.1-eksbuild.8      ║
║ 1.33          ║ v1.21.1-eksbuild.8      ║
║ 1.32          ║ v1.21.1-eksbuild.8      ║
║ 1.31          ║ v1.21.1-eksbuild.8      ║
║ 1.30          ║ v1.21.1-eksbuild.8      ║
║ 1.29          ║ v1.21.1-eksbuild.8      ║
╚═══════════════╩═════════════════════════╝
```

---

## 2. eksctl — Config File YAML

[FATO] eksctl usa `apiVersion: eksctl.io/v1alpha5` e `kind: ClusterConfig`. A AWS recomenda usar config file em vez de flags inline em ambientes de produção — o YAML pode ser versionado no Git e o flag `--dry-run` permite inspecionar o que será criado antes de aplicar.

[FATO] O eksctl cria **managed node groups** por padrão (desde eksctl 0.31+). Para self-managed, usar `--managed=false` ou especificar `nodeGroups:` (ao invés de `managedNodeGroups:`) no YAML.

### 2.1 Config mínimo — cluster com managed node group

```yaml
# cluster-basic.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: my-cluster
  region: us-east-1
  version: "1.33"         # versão do Kubernetes — entre aspas (string, não número)
  tags:
    project: checkout
    env: prod

# IAM OIDC provider — necessário para IRSA (IAM Roles for Service Accounts)
iam:
  withOIDC: true

managedNodeGroups:
  - name: ng-workers
    instanceType: m5.xlarge
    desiredCapacity: 3
    minSize: 2
    maxSize: 10
    volumeSize: 100        # GB, gp3 por padrão
    privateNetworking: true  # nodes em subnets privadas (recomendado)
    labels:
      role: worker
    tags:
      k8s.io/cluster-autoscaler/enabled: "true"
      k8s.io/cluster-autoscaler/my-cluster: "owned"
    iam:
      withAddonPolicies:
        autoScaler: true          # policy para Cluster Autoscaler
        albIngress: true          # policy para AWS Load Balancer Controller
        cloudWatch: true          # policy para Container Insights
```

### 2.2 Config completo — cluster em VPC existente, múltiplos node groups

```yaml
# cluster-production.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: checkout-prod
  region: us-east-1
  version: "1.33"
  tags:
    project: checkout
    env: prod
    cost-center: platform-engineering

# VPC existente (não criar nova VPC)
vpc:
  id: vpc-0abc12345def67890
  subnets:
    private:
      us-east-1a: { id: subnet-0111aaaa }
      us-east-1b: { id: subnet-0222bbbb }
      us-east-1c: { id: subnet-0333cccc }
    public:
      us-east-1a: { id: subnet-0444dddd }
      us-east-1b: { id: subnet-0555eeee }
      us-east-1c: { id: subnet-0666ffff }
  # Endpoints privados para o kube-apiserver
  clusterEndpoints:
    publicAccess: true    # acesso via internet (necessário para kubectl externo)
    privateAccess: true   # acesso interno na VPC (necessário para nodes se comunicarem)

iam:
  withOIDC: true

# Add-ons gerenciados
addons:
  - name: vpc-cni
    version: latest
    attachPolicyARNs:
      - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
    configurationValues: |-
      env:
        ENABLE_PREFIX_DELEGATION: "true"
        WARM_PREFIX_TARGET: "1"
  - name: kube-proxy
    version: latest
  - name: coredns
    version: latest
  - name: aws-ebs-csi-driver
    version: latest

# Node group principal — workloads da aplicação
managedNodeGroups:
  - name: ng-app-workers
    instanceTypes:
      - m5.xlarge
      - m5a.xlarge
      - m5n.xlarge       # múltiplos tipos aumentam disponibilidade de capacidade Spot
    spot: false           # On-Demand para carga estável
    desiredCapacity: 6
    minSize: 3
    maxSize: 30
    volumeSize: 100
    privateNetworking: true
    availabilityZones: [us-east-1a, us-east-1b, us-east-1c]
    labels:
      role: app-worker
      tier: application
    taints:
      - key: workload
        value: app
        effect: NoSchedule
    ssh:
      allow: false        # SSM Session Manager em vez de SSH direto

  # Node group para workloads de infra (monitoring, logging)
  - name: ng-infra
    instanceType: m5.large
    desiredCapacity: 2
    minSize: 2
    maxSize: 5
    privateNetworking: true
    labels:
      role: infra

# Fargate Profile para namespaces específicos
fargateProfiles:
  - name: fp-batch-jobs
    selectors:
      - namespace: batch
        labels:
          fargate: "true"
      - namespace: kube-system
        labels:
          k8s-app: kube-dns
```

### 2.3 Comandos essenciais

```bash
# Dry-run: gera ClusterConfig completo com defaults sem criar nada
eksctl create cluster -f cluster-production.yaml --dry-run

# Criar cluster (leva ~15-20 min)
eksctl create cluster -f cluster-production.yaml

# Verificar stacks CloudFormation criadas pelo eksctl
aws cloudformation list-stacks \
  --stack-status-filter CREATE_COMPLETE \
  --query 'StackSummaries[?starts_with(StackName, `eksctl-`)].{Name:StackName,Status:StackStatus}' \
  --output table

# Deletar cluster (usar --wait para aguardar conclusão completa)
eksctl delete cluster -f cluster-production.yaml --wait
```

[FATO] O eksctl cria os seguintes CloudFormation stacks:
- `eksctl-<name>-cluster` — VPC, subnets (se não VPC existente), control plane
- `eksctl-<name>-nodegroup-<ng-name>` — um stack por node group
- `eksctl-<name>-addon-iamserviceaccount-<namespace>-<sa>` — um por IRSA

---

## 3. kubeconfig — Autenticação com aws eks update-kubeconfig

[FATO] O EKS usa o plugin de autenticação `aws eks get-token` (integrado na AWS CLI v2 como exec credential plugin). O token tem validade de 15 minutos e é renovado automaticamente pelo kubectl quando necessário.

```bash
# Configurar kubeconfig (cria ou mescla com ~/.kube/config existente)
aws eks update-kubeconfig \
  --region us-east-1 \
  --name checkout-prod

# Usar role IAM específica para autenticação (útil em CI/CD)
aws eks update-kubeconfig \
  --region us-east-1 \
  --name checkout-prod \
  --role-arn arn:aws:iam::123456789012:role/EKSDeployRole

# Salvar em arquivo separado (não mesclar com ~/.kube/config)
aws eks update-kubeconfig \
  --region us-east-1 \
  --name checkout-prod \
  --kubeconfig ~/.kube/checkout-prod.yaml

# Verificar identidade sendo usada (antes de executar kubectl)
aws sts get-caller-identity

# Verificar configuração do kubeconfig
kubectl config view --minify

# Testar conectividade com o cluster
kubectl get svc
```

[FATO] O arquivo `~/.kube/config` gerado pelo `update-kubeconfig` contém um bloco de exec credentials como este:

```yaml
users:
- name: arn:aws:eks:us-east-1:123456789012:cluster/checkout-prod
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      command: aws
      args:
        - eks
        - get-token
        - --cluster-name
        - checkout-prod
        - --region
        - us-east-1
```

[FATO] Apenas o principal IAM que criou o cluster tem acesso inicial ao Kubernetes RBAC (como cluster-admin). Para adicionar outros principals, é necessário criar um `EKS Access Entry` (API moderna) ou editar o ConfigMap `aws-auth` (método legado, deprecado em versões recentes).

```bash
# Método moderno (EKS API, recomendado a partir de EKS 1.28)
aws eks create-access-entry \
  --cluster-name checkout-prod \
  --principal-arn arn:aws:iam::123456789012:role/DevOpsRole \
  --type STANDARD

aws eks associate-access-policy \
  --cluster-name checkout-prod \
  --principal-arn arn:aws:iam::123456789012:role/DevOpsRole \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster
```

---

## 4. VPC CNI — Modelo de IP Allocation

### 4.1 Secondary IP Mode (padrão)

[FATO] O Amazon VPC CNI (plugin `aws-node`, um DaemonSet) implementa o networking de pods. Em **Secondary IP Mode** (padrão):

```
┌─────────────────────────────────────────────────────────────┐
│  EC2 Node (ex.: m5.xlarge)                                  │
│                                                             │
│  ENI 0 (primary ENI)                                        │
│  ├── 10.0.1.10  ← IP primário do node                      │
│  ├── 10.0.1.11  ← secondary IP → Pod A                     │
│  ├── 10.0.1.12  ← secondary IP → Pod B                     │
│  └── ...até max IPs por ENI                                 │
│                                                             │
│  ENI 1 (secondary ENI — warm pool)                          │
│  ├── 10.0.1.25  ← secondary IP → Pod C                     │
│  ├── 10.0.1.26  ← secondary IP (warm — aguardando pod)     │
│  └── ...                                                    │
│                                                             │
│  ENI 2 (secondary ENI — warm ENI)                           │
│  ├── 10.0.1.40  ← secondary IP (warm)                      │
│  └── ...                                                    │
│                                                             │
│  DaemonSet aws-node (ipamd):                                │
│  - gerencia ENIs e IPs                                      │
│  - mantém warm pool                                         │
│  - invocado pelo kubelet via CNI binary                     │
└─────────────────────────────────────────────────────────────┘
```

[FATO] Cada Pod recebe **um IP do VPC** — não há NAT entre pods e a rede VPC. Um Pod com IP 10.0.1.11 é acessível diretamente de qualquer recurso no VPC (EC2, RDS, Lambda) usando esse IP.

### 4.2 Fórmula de pod density (Secondary IP Mode)

[FATO] Fórmula oficial AWS para o número máximo de Pods por node:

```
max_pods = (ENIs × (IPs_per_ENI - 1)) + 2

O +2 representa kube-proxy e VPC CNI que rodam em hostNetwork
(não consomem secondary IP)
```

[FATO] Exemplos de instance types comuns (fonte: aws/amazon-vpc-cni-k8s/misc/eni-max-pods.txt):

```
╔═══════════════╦══════╦════════════╦═══════════╦═════════════════╗
║ Instance Type ║ vCPU ║ Max ENIs   ║ IPs/ENI   ║ Max Pods (fórmula) ║
╠═══════════════╬══════╬════════════╬═══════════╬═════════════════╣
║ t3.small      ║   2  ║     3      ║     4     ║ 3×(4-1)+2 = 11  ║
║ t3.medium     ║   2  ║     3      ║     6     ║ 3×(6-1)+2 = 17  ║
║ t3.large      ║   2  ║     3      ║    12     ║ 3×(12-1)+2 = 35 ║
║ m5.large      ║   2  ║     3      ║    10     ║ 3×(10-1)+2 = 29 ║
║ m5.xlarge     ║   4  ║     4      ║    15     ║ 4×(15-1)+2 = 58 ║
║ m5.2xlarge    ║   8  ║     4      ║    15     ║ 4×(15-1)+2 = 58 ║
║ m5.4xlarge    ║  16  ║     8      ║    30     ║ 8×(30-1)+2 = 234║
║ c5.large      ║   2  ║     3      ║    10     ║ 3×(10-1)+2 = 29 ║
║ c5.xlarge     ║   4  ║     4      ║    15     ║ 4×(15-1)+2 = 58 ║
║ c5.2xlarge    ║   8  ║     4      ║    15     ║ 4×(15-1)+2 = 58 ║
║ c5.4xlarge    ║  16  ║     8      ║    30     ║ 8×(30-1)+2 = 234║
╚═══════════════╩══════╩════════════╩═══════════╩═════════════════╝
```

[FATO] O limite de 110 Pods por node é o **máximo padrão do kubelet**, não o limite do VPC CNI. Instâncias grandes (ex.: m5.4xlarge com 234 slots via fórmula) são limitadas a 110 pelo kubelet a menos que `--max-pods` seja configurado explicitamente. Com **Prefix Delegation**, é possível ultrapassar 110 em instâncias grandes.

### 4.3 Warm Pool — variáveis de controle

[FATO] O ipamd mantém um warm pool de IPs/ENIs prontos para pods que ainda não foram agendados. As variáveis de controle (configuradas no DaemonSet `aws-node`):

```
╔════════════════════╦═══════════════════════════════════════════════╗
║ Variável           ║ Significado                                   ║
╠════════════════════╬═══════════════════════════════════════════════╣
║ WARM_ENI_TARGET    ║ Número de ENIs "warm" (sem pods) mantidas     ║
║ (padrão: 1)        ║ além das ENIs em uso. Cada ENI warm consome   ║
║                    ║ IPs do CIDR da subnet mesmo sem pods.         ║
╠════════════════════╬═══════════════════════════════════════════════╣
║ WARM_IP_TARGET     ║ Número de IPs warm disponíveis. Sobreescreve  ║
║                    ║ WARM_ENI_TARGET quando configurado.           ║
╠════════════════════╬═══════════════════════════════════════════════╣
║ MINIMUM_IP_TARGET  ║ Mínimo de IPs alocados em qualquer momento.   ║
║                    ║ Front-loads ENIs no launch do node.           ║
╚════════════════════╩═══════════════════════════════════════════════╝
```

[FATO] IPs "warm" ainda consomem endereços do CIDR da subnet mesmo sem pods associados. Em subnets com /24 (254 IPs), um node group de 10 nodes com WARM_ENI_TARGET=1 pode consumir até 50-100 IPs só de warm pool.

### 4.4 Prefix Delegation Mode (alternativa)

[FATO] Em vez de alocar IPs individuais como secondary IPs, o VPC CNI pode alocar um **prefixo /28** (16 IPs) por slot em cada ENI. Isso multiplica a pod density por ~16×.

```
Secondary IP Mode:
  ENI com 15 IPs/ENI → 14 pods por ENI

Prefix Delegation Mode:
  ENI com 15 prefixes/ENI × 16 IPs cada = 240 IPs → muito mais pods por ENI
```

[FATO] Prefix Delegation requer instâncias **Nitro** (não funciona em instâncias Xen como t2, m4 anteriores). Habilitado via variável de ambiente `ENABLE_PREFIX_DELEGATION=true` no DaemonSet `aws-node` ou via managed add-on configuration.

[FATO] A transição de Secondary IP → Prefix Delegation deve ser feita criando um **novo node group** — não fazer rolling update de nodes existentes (pode causar inconsistência na contagem de IPs disponíveis reportada ao scheduler).

---

## 5. CDK Python — Cluster EKS

```python
from aws_cdk import (
    Stack, CfnOutput,
    aws_eks as eks,
    aws_ec2 as ec2,
    aws_iam as iam,
)
from constructs import Construct


class EksClusterStack(Stack):
    def __init__(self, scope: Construct, construct_id: str,
                 vpc: ec2.IVpc, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # ──────────────────────────────────────────────────────────────
        # IAM role para o control plane
        # ──────────────────────────────────────────────────────────────
        cluster_role = iam.Role(self, "ClusterRole",
            assumed_by=iam.ServicePrincipal("eks.amazonaws.com"),
            managed_policies=[
                iam.ManagedPolicy.from_aws_managed_policy_name("AmazonEKSClusterPolicy"),
            ],
        )

        # ──────────────────────────────────────────────────────────────
        # IAM role para os nodes (Node Instance Role)
        # ──────────────────────────────────────────────────────────────
        node_role = iam.Role(self, "NodeRole",
            assumed_by=iam.ServicePrincipal("ec2.amazonaws.com"),
            managed_policies=[
                iam.ManagedPolicy.from_aws_managed_policy_name("AmazonEKSWorkerNodePolicy"),
                iam.ManagedPolicy.from_aws_managed_policy_name("AmazonEC2ContainerRegistryReadOnly"),
                # AmazonEKS_CNI_Policy: necessário para o VPC CNI
                # ATENÇÃO: recomendado mover para IRSA do aws-node DaemonSet em produção
                iam.ManagedPolicy.from_aws_managed_policy_name("AmazonEKS_CNI_Policy"),
                # Para Container Insights
                iam.ManagedPolicy.from_aws_managed_policy_name("CloudWatchAgentServerPolicy"),
            ],
        )

        # ──────────────────────────────────────────────────────────────
        # Cluster EKS (L2 construct)
        # ──────────────────────────────────────────────────────────────
        cluster = eks.Cluster(self, "CheckoutCluster",
            cluster_name="checkout-prod",
            version=eks.KubernetesVersion.V1_33,
            role=cluster_role,
            vpc=vpc,
            vpc_subnets=[ec2.SubnetSelection(subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS)],
            # Endpoint access: público + privado (recomendado)
            endpoint_access=eks.EndpointAccess.PUBLIC_AND_PRIVATE,
            # OIDC provider habilitado automaticamente (para IRSA)
            # default_capacity=0: não criar node group padrão
            # (vamos criar managed node group customizado abaixo)
            default_capacity=0,
            kubectl_layer=None,   # usa layer padrão
        )

        # ──────────────────────────────────────────────────────────────
        # Managed Node Group
        # ──────────────────────────────────────────────────────────────
        node_group = cluster.add_nodegroup_capacity("AppWorkers",
            nodegroup_name="ng-app-workers",
            instance_types=[
                ec2.InstanceType("m5.xlarge"),
                ec2.InstanceType("m5a.xlarge"),
                ec2.InstanceType("m5n.xlarge"),
            ],
            min_size=3,
            desired_size=6,
            max_size=30,
            disk_size=100,
            node_role=node_role,
            # AMI type padrão: AL2_x86_64 (Amazon Linux 2)
            ami_type=eks.NodegroupAmiType.AL2_X86_64,
            capacity_type=eks.CapacityType.ON_DEMAND,
            labels={"role": "app-worker", "tier": "application"},
            tags={"project": "checkout", "env": "prod"},
        )

        # ──────────────────────────────────────────────────────────────
        # VPC CNI Managed Add-on com Prefix Delegation
        # ──────────────────────────────────────────────────────────────
        # L2 de add-ons usa CfnAddon (L1)
        from aws_cdk import aws_eks as eks_l1
        vpc_cni_addon = eks.CfnAddon(self, "VpcCniAddon",
            cluster_name=cluster.cluster_name,
            addon_name="vpc-cni",
            # Sempre usar latest ou versão específica compatível com K8s version
            resolve_conflicts_on_update="OVERWRITE",
            configuration_values='{"env":{"ENABLE_PREFIX_DELEGATION":"true","WARM_PREFIX_TARGET":"1"}}',
        )
        # O add-on só pode ser criado após o cluster existir
        vpc_cni_addon.node.add_dependency(cluster)

        # ──────────────────────────────────────────────────────────────
        # Outputs
        # ──────────────────────────────────────────────────────────────
        CfnOutput(self, "ClusterName",
            value=cluster.cluster_name,
            description="Nome do cluster EKS",
        )
        CfnOutput(self, "ClusterEndpoint",
            value=cluster.cluster_endpoint,
        )
        CfnOutput(self, "OidcIssuer",
            value=cluster.cluster_open_id_connect_issuer,
            description="OIDC issuer URL para IRSA",
        )
        CfnOutput(self, "UpdateKubeconfigCmd",
            value=f"aws eks update-kubeconfig --region {self.region} --name {cluster.cluster_name}",
            description="Comando para configurar kubectl",
        )
```

---

## 6. Python — Script de verificação de pod density por instance type

```python
"""
Calcula pod density por instance type usando a fórmula VPC CNI.
Útil para planejar capacity antes de criar node groups.
"""
from dataclasses import dataclass

# Limites EC2 por instance type (ENIs e IPs/ENI)
# Fonte: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html
# (tabela "Network Interfaces per Instance Type")
EC2_NETWORK_LIMITS: dict[str, tuple[int, int]] = {
    # (max_enis, max_ips_per_eni)
    "t3.small":    (3,  4),
    "t3.medium":   (3,  6),
    "t3.large":    (3, 12),
    "t3.xlarge":   (4, 15),
    "t3.2xlarge":  (4, 15),
    "m5.large":    (3, 10),
    "m5.xlarge":   (4, 15),
    "m5.2xlarge":  (4, 15),
    "m5.4xlarge":  (8, 30),
    "m5.8xlarge":  (8, 30),
    "m5.12xlarge": (8, 30),
    "m5.16xlarge": (15, 50),
    "m5.24xlarge": (15, 50),
    "c5.large":    (3, 10),
    "c5.xlarge":   (4, 15),
    "c5.2xlarge":  (4, 15),
    "c5.4xlarge":  (8, 30),
    "c5.9xlarge":  (8, 30),
    "c5.12xlarge": (12, 30),
    "r5.large":    (3, 10),
    "r5.xlarge":   (4, 15),
    "r5.2xlarge":  (4, 15),
    "r5.4xlarge":  (8, 30),
}

KUBELET_MAX_PODS_DEFAULT = 110   # limite padrão do kubelet


@dataclass
class PodDensityResult:
    instance_type: str
    max_enis: int
    ips_per_eni: int
    max_pods_secondary_ip: int       # fórmula: (ENIs × (IPs/ENI - 1)) + 2
    max_pods_prefix_delegation: int  # prefix /28 = 16 IPs por slot
    max_pods_effective: int          # limitado pelo kubelet a 110 (sem configuração extra)
    is_kubelet_limited: bool


def calc_pod_density(instance_type: str, override_max_pods: int | None = None) -> PodDensityResult:
    """Calcula pod density para um instance type."""
    if instance_type not in EC2_NETWORK_LIMITS:
        raise ValueError(f"Instance type '{instance_type}' not in local database. "
                         "Check https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html")

    max_enis, ips_per_eni = EC2_NETWORK_LIMITS[instance_type]

    # Fórmula oficial VPC CNI (Secondary IP Mode)
    max_pods_secondary = (max_enis * (ips_per_eni - 1)) + 2

    # Prefix Delegation: /28 = 16 IPs por slot (prefixo)
    # Cada slot que antes tinha 1 IP agora pode ter 16
    # slots por ENI = ips_per_eni (é o mesmo limite de "slots")
    max_pods_prefix = (max_enis * (ips_per_eni - 1) * 16) + 2

    effective_limit = override_max_pods or KUBELET_MAX_PODS_DEFAULT
    max_pods_effective = min(max_pods_secondary, effective_limit)
    is_limited = max_pods_secondary > effective_limit

    return PodDensityResult(
        instance_type=instance_type,
        max_enis=max_enis,
        ips_per_eni=ips_per_eni,
        max_pods_secondary_ip=max_pods_secondary,
        max_pods_prefix_delegation=max_pods_prefix,
        max_pods_effective=max_pods_effective,
        is_kubelet_limited=is_limited,
    )


def print_density_report(instance_types: list[str]) -> None:
    """Imprime relatório comparativo de pod density."""
    print(f"\n{'Instance Type':15} {'ENIs':5} {'IPs/ENI':7} "
          f"{'SecIP Pods':10} {'Prefix Pods':11} {'Effective':9} {'Kubelet Limit?':14}")
    print("-" * 75)

    for itype in instance_types:
        try:
            r = calc_pod_density(itype)
            limited = "⚠ YES" if r.is_kubelet_limited else "No"
            print(f"{r.instance_type:15} {r.max_enis:5} {r.ips_per_eni:7} "
                  f"{r.max_pods_secondary_ip:10} {r.max_pods_prefix_delegation:11} "
                  f"{r.max_pods_effective:9} {limited:14}")
        except ValueError as e:
            print(f"{itype:15} ERROR: {e}")

    print(f"\nNote: 'Effective' = min(SecIP Pods, {KUBELET_MAX_PODS_DEFAULT})")
    print("      To exceed 110, set --max-pods in kubelet or use Prefix Delegation + override")


def calc_subnet_ip_consumption(
    instance_type: str,
    node_count: int,
    warm_eni_target: int = 1,
) -> dict:
    """Estima consumo de IPs de subnet para um node group."""
    if instance_type not in EC2_NETWORK_LIMITS:
        raise ValueError(f"Unknown instance type: {instance_type}")

    max_enis, ips_per_eni = EC2_NETWORK_LIMITS[instance_type]

    # IPs mínimos: cada node tem primary ENI + IPs de pods
    # IPs máximos: cada node tem max_enis ENIs × ips_per_eni (incluindo warm pool)
    ips_per_node_min = ips_per_eni             # primary ENI com IPs
    ips_per_node_max = max_enis * ips_per_eni  # todos ENIs totalmente alocados (warm pool)

    return {
        "instance_type": instance_type,
        "node_count": node_count,
        "warm_eni_target": warm_eni_target,
        "ips_consumed_min": ips_per_node_min * node_count,
        "ips_consumed_max": ips_per_node_max * node_count,
        "subnet_24_ips_available": 254,
        "fits_in_24": ips_per_node_max * node_count <= 254,
        "recommendation": (
            "Use /22 or larger subnet for this node group"
            if ips_per_node_max * node_count > 254
            else "A /24 subnet is sufficient"
        ),
    }


if __name__ == "__main__":
    INSTANCE_TYPES = [
        "t3.medium", "t3.large",
        "m5.large", "m5.xlarge", "m5.2xlarge", "m5.4xlarge",
        "c5.large", "c5.xlarge", "c5.4xlarge",
    ]

    print_density_report(INSTANCE_TYPES)

    # Planejamento de subnet para m5.xlarge com 10 nodes
    consumption = calc_subnet_ip_consumption("m5.xlarge", node_count=10)
    print(f"\nSubnet IP consumption for {consumption['node_count']} × {consumption['instance_type']}:")
    print(f"  Min IPs (pods at minimum): {consumption['ips_consumed_min']}")
    print(f"  Max IPs (warm pool fully allocated): {consumption['ips_consumed_max']}")
    print(f"  Recommendation: {consumption['recommendation']}")
```

---

## 7. CLI — Verificação e Diagnóstico do Cluster

```bash
# ─────────────────────────────────────────────────────────
# APÓS o eksctl/CDK criar o cluster, configurar kubectl:
# ─────────────────────────────────────────────────────────

aws eks update-kubeconfig --region us-east-1 --name checkout-prod

# ─────────────────────────────────────────────────────────
# Verificar nodes (Ready = node pronto para receber pods)
# ─────────────────────────────────────────────────────────

kubectl get nodes -o wide
# NAME                        STATUS  ROLES  AGE  VERSION      INTERNAL-IP  INSTANCE-ID
# ip-10-0-1-10.us-east-1...  Ready  <none>  5m  v1.33.0-eks  10.0.1.10    i-0abc123

# Verificar detalhes de capacidade do node (allocatable pods)
kubectl describe node ip-10-0-1-10.us-east-1.compute.internal | grep -A5 "Allocatable"
# Allocatable:
#   cpu:     3920m
#   memory:  15168Mi
#   pods:    58           ← max pods do node (calculado pelo VPC CNI)

# ─────────────────────────────────────────────────────────
# Verificar pods do sistema (incluindo VPC CNI e kube-proxy)
# ─────────────────────────────────────────────────────────

kubectl get pods -n kube-system -o wide
# NAMESPACE     NAME              READY   STATUS    NODE
# kube-system   aws-node-xxxxx    1/1     Running   ip-10-0-1-10...   ← VPC CNI DaemonSet
# kube-system   kube-proxy-xxxxx  1/1     Running   ip-10-0-1-10...
# kube-system   coredns-xxxxx     1/1     Running   ip-10-0-1-10...

# ─────────────────────────────────────────────────────────
# Verificar IP allocation no node (ENIs e IPs alocados)
# ─────────────────────────────────────────────────────────

# Listar ENIs do node (via EC2 API — substitua o node IP)
NODE_IP="10.0.1.10"
aws ec2 describe-network-interfaces \
  --filters "Name=private-ip-address,Values=${NODE_IP}" \
  --query 'NetworkInterfaces[0].Attachment.InstanceId' \
  --output text | xargs -I {} aws ec2 describe-instances \
    --instance-ids {} \
    --query 'Reservations[0].Instances[0].NetworkInterfaces[*].{ENI:NetworkInterfaceId,IPs:PrivateIpAddresses[*].PrivateIpAddress}' \
    --output table

# Verificar add-on VPC CNI instalado
aws eks describe-addon \
  --cluster-name checkout-prod \
  --addon-name vpc-cni \
  --query 'addon.{Version:addonVersion,Status:status,Config:configurationValues}'

# ─────────────────────────────────────────────────────────
# Verificar configuração do aws-node DaemonSet
# ─────────────────────────────────────────────────────────

kubectl get daemonset aws-node -n kube-system -o jsonpath='{.spec.template.spec.containers[0].env}' | python3 -m json.tool

# Verificar variáveis de warm pool
kubectl get daemonset aws-node -n kube-system \
  -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="WARM_ENI_TARGET")].value}'

# ─────────────────────────────────────────────────────────
# Verificar Access Entries (autenticação IAM → K8s RBAC)
# ─────────────────────────────────────────────────────────

aws eks list-access-entries --cluster-name checkout-prod \
  --query 'accessEntries[*]' --output table

# Verificar quem tem acesso ao cluster
aws eks list-associated-access-policies \
  --cluster-name checkout-prod \
  --principal-arn arn:aws:iam::123456789012:role/DevOpsRole

# ─────────────────────────────────────────────────────────
# Troubleshooting: node não aparece como Ready
# ─────────────────────────────────────────────────────────

# 1. Verificar se node consegue se comunicar com o API server
kubectl get events --sort-by='.lastTimestamp' | tail -20

# 2. Verificar logs do aws-node (VPC CNI)
kubectl logs -n kube-system -l k8s-app=aws-node --tail=50

# 3. Verificar node conditions
kubectl describe node <node-name> | grep -A20 "Conditions:"

# 4. Verificar se a node role tem as policies necessárias
aws iam list-attached-role-policies \
  --role-name <node-role-name> \
  --query 'AttachedPolicies[*].PolicyName' \
  --output table
```

---

## 8. Armadilhas (Pitfalls)

[FATO] **Principal IAM que cria o cluster é o único com acesso inicial**: se o cluster for criado por um role de CI/CD e você tentar acessar com sua identidade pessoal, receberá `Unauthorized`. Solução: adicionar sua identidade como `EKS Access Entry` antes de precisar dela, ou criar o cluster com uma role compartilhada.

[FATO] **`aws-auth` ConfigMap vs EKS Access Entries**: versões antigas do EKS usavam o ConfigMap `aws-auth` para mapear IAM → K8s RBAC. A partir de EKS 1.28+, a API de Access Entries é o método recomendado. O ConfigMap ainda funciona mas está em modo legado — editar manualmente o `aws-auth` pode tornar o cluster inacessível se houver erro de YAML.

[CONSENSO] **Subnets muito pequenas são a causa mais comum de falha de scaling**: cada node consome vários IPs do CIDR da subnet (devido ao warm pool). Use subnets /22 ou maiores para subnets de nodes em produção. O erro `InsufficientFreeAddressesInSubnet` só aparece durante scaling, não na criação do cluster.

[FATO] **eksctl cria VPC nova por padrão**: se você não especificar `vpc:` no config, o eksctl cria uma nova VPC com CIDR 192.168.0.0/16. Em ambientes corporativos, sempre especifique VPC existente para evitar conflito de CIDR e custos de NAT Gateway duplicados.

[FATO] **Prefix Delegation não é retroativa em nodes existentes**: ativar `ENABLE_PREFIX_DELEGATION=true` no DaemonSet não muda a alocação dos nodes já em execução. Somente novos nodes (rolling update do node group) adotarão o novo modo.

[FATO] **kubectl version compatibility**: kubectl pode ser no máximo 1 minor version acima ou abaixo da versão do cluster. Um cluster 1.33 pode usar kubectl 1.32, 1.33 ou 1.34. Usar kubectl 1.30 com cluster 1.33 pode funcionar mas não é suportado oficialmente.

---

## Exercício de Reflexão

Você está planejando um cluster EKS para um serviço de processamento de imagens com as seguintes características:
- 3 microserviços: API (stateless, alta variância de carga), Worker (CPU-intensivo, carga estável), Redis Sidecar (memory-intensive)
- Requisito: cada Pod do Worker precisa de 3 vCPUs e 6 GB RAM
- SLA: 0 downtime durante atualizações de node group
- A subnet privada disponível é uma /24 (254 IPs)
- Budget: reduzir custos sem sacrificar disponibilidade

**Responda:**

1. Qual instance type escolher para os Workers (considerando 3 vCPU/6GB por pod e pod density)? Qual o máximo de Worker pods por node?
2. Quantos nodes cabem na subnet /24 sem esgotar IPs (considerando warm pool com WARM_ENI_TARGET=1)?
3. Como garantir 0 downtime durante rolling update do node group? Qual Kubernetes feature e quais configurações do node group são necessárias?
4. O VPC CNI com Secondary IP Mode é suficiente para essa subnet, ou vale habilitar Prefix Delegation? Justifique com cálculos.
5. A API (alta variância de carga) deve ficar no mesmo node group que os Workers? Por quê?

---

## Referências

- [FATO] [Get started with Amazon EKS – eksctl](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html) — docs.aws.amazon.com
- [FATO] [Creating and managing clusters – Eksctl User Guide](https://docs.aws.amazon.com/eks/latest/eksctl/creating-and-managing-clusters.html) — docs.aws.amazon.com
- [FATO] [Connect kubectl to an EKS cluster by creating a kubeconfig file](https://docs.aws.amazon.com/eks/latest/userguide/create-kubeconfig.html) — docs.aws.amazon.com
- [FATO] [Amazon VPC CNI – Best Practices Guide](https://docs.aws.amazon.com/eks/latest/best-practices/vpc-cni.html) — docs.aws.amazon.com
- [FATO] [Assign more IP addresses to Amazon EKS nodes with prefixes](https://docs.aws.amazon.com/eks/latest/userguide/cni-increase-ip-addresses.html) — docs.aws.amazon.com
- [FATO] [Assign IPs to Pods with the Amazon VPC CNI](https://docs.aws.amazon.com/eks/latest/userguide/managing-vpc-cni.html) — docs.aws.amazon.com
