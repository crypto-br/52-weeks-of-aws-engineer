# Sessão 047 — EKS: Managed Node Groups vs Fargate, Upgrades e Node Drain

**Duração estimada:** 60 min  
**Pré-requisito:** session-046 (EKS cluster provisioning, eksctl, VPC CNI)

---

## Objetivos da sessão

- Configurar managed node groups com labels e taints para separar workloads por perfil
- Criar Fargate Profiles com selectors por namespace/labels e entender todas as suas limitações
- Executar cluster version upgrade na ordem correta (control plane → add-ons → node groups)
- Entender as 4 fases do managed node group update (Setup, Scale Up, Upgrade, Scale Down)
- Executar `kubectl drain` em um node sem downtime de workload

---

## 1. Managed Node Groups — Conceitos Fundamentais

[FATO] Todo managed node group é implementado como um **EC2 Auto Scaling Group** gerenciado pelo EKS, rodando dentro da conta do cliente. O EKS provê a API de alto nível; os recursos EC2 e ASG estão na conta do cliente e são visíveis no console EC2.

[FATO] O EKS aplica automaticamente os seguintes labels a cada node de um managed node group (prefixados com `eks.amazonaws.com/`):

```
eks.amazonaws.com/nodegroup=<nome-do-nodegroup>
eks.amazonaws.com/nodegroup-image=<ami-id>
eks.amazonaws.com/capacityType=ON_DEMAND  # ou SPOT
```

[FATO] Labels e taints adicionais podem ser aplicados e **atualizados** via `update-nodegroup-config` sem precisar recriar o node group. A atualização de labels/taints é aplicada a **todos os nodes novos**; nodes existentes recebem a atualização na próxima rolling update (quando o AMI é atualizado).

### 1.1 Separação de workloads com labels e taints

```
Strategy: múltiplos node groups por perfil de workload

┌──────────────────────────────────────────────────────────────────┐
│  EKS Cluster                                                     │
│                                                                  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │ ng-app-ondemand │  │  ng-spot-batch  │  │  ng-gpu-ml      │  │
│  │                 │  │                 │  │                 │  │
│  │ m5.xlarge       │  │ c5.xlarge +     │  │ p3.2xlarge      │  │
│  │ capacity: OD    │  │ c5a/n.xlarge    │  │ capacity: OD    │  │
│  │                 │  │ capacity: SPOT  │  │                 │  │
│  │ label:          │  │ label:          │  │ label:          │  │
│  │  tier=app       │  │  tier=batch     │  │  tier=ml        │  │
│  │                 │  │                 │  │  accelerator=gpu│  │
│  │ taint:          │  │ taint:          │  │                 │  │
│  │  (nenhum)       │  │  tier=batch     │  │ taint:          │  │
│  │                 │  │  :NoSchedule   │  │  tier=ml        │  │
│  └─────────────────┘  └─────────────────┘  │  :NoSchedule   │  │
│                                             └─────────────────┘  │
│  Workload app:          Workload batch:      Workload ML:         │
│  nodeSelector:          nodeSelector:        nodeSelector:        │
│    tier: app              tier: batch          accelerator: gpu   │
│                         tolerations:          tolerations:        │
│                         - key: tier             - key: tier       │
│                           value: batch            value: ml       │
│                           effect: NoSchedule      effect: NOS..  │
└──────────────────────────────────────────────────────────────────┘
```

### 1.2 Capacity Types: On-Demand e Spot em node groups

[FATO] Um managed node group só pode ser **ON_DEMAND ou SPOT** — não é possível misturar os dois tipos em um mesmo node group. Para workloads que usam os dois, crie dois node groups separados (um OD e um Spot).

[FATO] Para Spot, a allocation strategy é:
- `price-capacity-optimized` (PCO) — clusters K8s **1.28+** (mais recente, recomendado)
- `capacity-optimized` (CO) — clusters K8s **1.27 e anteriores** (não muda para node groups existentes)

[FATO] Managed node groups Spot têm **Capacity Rebalancing habilitado** por padrão: quando um Spot node recebe Rebalance Recommendation, o EKS tenta lançar um node substituto antes de drenar o original.

---

## 2. Fargate Profiles — Seletores e Limitações

### 2.1 Modelo de execução Fargate

[FATO] Cada Pod no Fargate tem **sua própria VM dedicada** (kernel, CPU, memória e ENI isolados). Não há host compartilhado entre Pods. Isso oferece isolamento mais forte que EC2, porém remove capacidades que dependem do host.

[FATO] No Fargate, **cada Pod = um node** em `kubectl get nodes`. O nome do node segue o padrão `fargate-ip-<ip>.region.compute.internal`.

```bash
kubectl get nodes
# NAME                                                       STATUS  VERSION
# fargate-ip-10-0-1-20.us-east-1.compute.internal           Ready   v1.33.0-eks-xxx
# fargate-ip-10-0-1-21.us-east-1.compute.internal           Ready   v1.33.0-eks-xxx
# ip-10-0-2-10.us-east-1.compute.internal                   Ready   v1.33.0-eks-xxx  ← EC2 node
```

### 2.2 Limitações do Fargate (tabela completa)

[FATO] As seguintes limitações se aplicam a Fargate no EKS:

```
╔═══════════════════════════════════════════════╦═══════╦═════════════════════════════╗
║ Recurso / Capacidade                          ║ Suport║ Alternativa                 ║
╠═══════════════════════════════════════════════╬═══════╬═════════════════════════════╣
║ DaemonSets                                    ║  NÃO  ║ Sidecar container           ║
║ Privileged containers                         ║  NÃO  ║ Usar EC2 node group         ║
║ HostPort / HostNetwork                        ║  NÃO  ║ Service ClusterIP ou LB     ║
║ GPUs                                          ║  NÃO  ║ EC2 node group (p3, g4dn)   ║
║ Amazon EBS (volumes)                          ║  NÃO  ║ Amazon EFS (auto-mount)     ║
║ Public subnets                                ║  NÃO  ║ Private subnet + NAT GW     ║
║ Spot capacity type                            ║  NÃO  ║ Managed node group Spot     ║
║ Custom AMI                                    ║  NÃO  ║ EC2 node group              ║
║ Custom CNI                                    ║  NÃO  ║ EC2 node group              ║
║ SSH access                                    ║  NÃO  ║ kubectl exec                ║
║ AWS Outposts / Wavelength / Local Zones       ║  NÃO  ║ EC2 node group              ║
║ Windows containers                            ║  NÃO  ║ EC2 Windows node group      ║
║ ARM / Graviton                                ║  NÃO  ║ EC2 Graviton node group     ║
║ IMDS (instance metadata)                      ║  NÃO  ║ IRSA para credenciais IAM   ║
╠═══════════════════════════════════════════════╬═══════╬═════════════════════════════╣
║ Amazon EFS (static provisioning)              ║  SIM  ║ —                           ║
║ ALB / NLB (target type IP)                    ║  SIM  ║ — (não usa node IP mode)    ║
║ Security Groups per Pod (via IRSA/SG for Pods)║  SIM  ║ —                           ║
║ VPA / HPA                                     ║  SIM  ║ —                           ║
╚═══════════════════════════════════════════════╩═══════╩═════════════════════════════╝
```

[FATO] **IMDS não está disponível** para Pods no Fargate. Aplicações que precisam de credenciais IAM **devem usar IRSA** (IAM Roles for Service Accounts). Aplicações que acessam IMDS para obter Region ou AZ devem ter esses valores hard-coded no Pod spec ou via `downwardAPI`.

### 2.3 Fargate Profile — estrutura

[FATO] Cada Fargate Profile pode ter até **5 selectors**. Cada selector exige namespace (campo obrigatório) e pode ter labels opcionais. Um Pod deve corresponder a pelo menos um selector para ser agendado no Fargate.

```yaml
# Fargate profile via eksctl ClusterConfig
fargateProfiles:
  - name: fp-apps
    selectors:
      # Selector 1: qualquer pod no namespace "analytics" com label fargate=true
      - namespace: analytics
        labels:
          fargate: "true"
      # Selector 2: qualquer pod no namespace "batch" (sem filtro de label)
      - namespace: batch
      # Selector 3: pods do CoreDNS no kube-system
      - namespace: kube-system
        labels:
          k8s-app: kube-dns
    # Subnets privadas para os pods Fargate (obrigatório)
    subnets:
      - subnet-0111aaaa
      - subnet-0222bbbb
      - subnet-0333cccc
```

[FATO] Pods que não correspondem a nenhum Fargate profile ficam no estado **Pending** indefinidamente (não são agendados nem em Fargate nem em EC2 automaticamente). Se houver um EC2 node group no cluster, o scheduler tentará colocar o Pod lá.

### 2.4 Sizing de Fargate Pods

[FATO] O Fargate provisiona um micro-VM com recursos iguais à soma dos requests de todos os containers no Pod, arredondado para cima para a próxima combinação suportada. Combinações disponíveis: 0.25–16 vCPU e 0.5–120 GB (com regras de proporção mínima).

[FATO] Use **Vertical Pod Autoscaler (VPA)** com mode `Auto` ou `Recreate` para ajustar o sizing de pods Fargate — o VPA recria o Pod com novos resource requests, e o Fargate provisiona a VM correspondente.

---

## 3. Cluster Version Upgrade — Ordem e Restrições

### 3.1 Regras fundamentais

[FATO] O upgrade do cluster EKS deve seguir a **ordem obrigatória**:

```
1. Control Plane  →  2. Add-ons  →  3. Node Groups  →  4. kubectl
    (EKS gerencia)     (VPC CNI,       (managed NG         (1 minor
                       CoreDNS,        rolling update)      version skew)
                       kube-proxy,
                       EBS CSI)
```

[FATO] O upgrade é **somente um minor version por vez** — não é possível saltar de 1.32 direto para 1.34. Cada step (1.32→1.33, 1.33→1.34) requer um upgrade separado.

[FATO] O upgrade do control plane é **irreversível** — não é possível fazer downgrade. Se precisar voltar para uma versão anterior, deve criar um novo cluster na versão desejada e migrar os workloads.

[FATO] A partir de K8s 1.28, o kubelet pode estar até **3 minor versions** atrás do kube-apiserver (skew policy upstream). Na prática, com kubelet 1.30 é possível fazer upgrades do control plane para 1.31, 1.32 e 1.33 antes de atualizar o node group.

[FATO] EKS requer até **5 IPs disponíveis** nas subnets do cluster para criar novas ENIs durante o upgrade do control plane. Se a subnet estiver cheia, o upgrade pode falhar.

### 3.2 Fluxo completo de upgrade

```
┌─────────────────────────────────────────────────────────────────────┐
│ Upgrade 1.32 → 1.33                                                  │
│                                                                      │
│ Step 1: Verificar estado do cluster                                  │
│   aws eks list-insights --cluster-name checkout-prod                 │
│   kubectl get nodes (todos devem estar em Ready + versão atual)      │
│                                                                      │
│ Step 2: Upgrade Control Plane                                        │
│   aws eks update-cluster-version --name checkout-prod                │
│   --kubernetes-version 1.33                                          │
│   [~15 min — rolling update de API servers, sem downtime]            │
│                                                                      │
│ Step 3: Upgrade Add-ons                                              │
│   aws eks update-addon --cluster-name checkout-prod                  │
│   --addon-name vpc-cni --resolve-conflicts OVERWRITE                 │
│   aws eks update-addon ... --addon-name coredns                      │
│   aws eks update-addon ... --addon-name kube-proxy                   │
│   aws eks update-addon ... --addon-name aws-ebs-csi-driver           │
│                                                                      │
│ Step 4: Upgrade Node Groups                                          │
│   aws eks update-nodegroup-version --cluster-name checkout-prod      │
│   --nodegroup-name ng-app-workers                                    │
│   [rolling update: Setup → Scale Up → Upgrade → Scale Down]         │
│                                                                      │
│ Step 5: Fargate Pods                                                 │
│   kubectl rollout restart deployment/my-app -n analytics             │
│   (Fargate pods existentes não são auto-atualizados)                 │
│                                                                      │
│ Step 6: Atualizar kubectl                                            │
│   brew upgrade kubectl  # ou equivalente                             │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.3 Fases do managed node group update

[FATO] Quando o EKS atualiza um managed node group (AMI update ou K8s version update), executa 4 fases:

```
┌─────────────────────────────────────────────────────────────────────┐
│ Fase 1: SETUP                                                        │
│   • Cria nova versão do EC2 Launch Template para o ASG               │
│   • Atualiza o ASG para usar a nova versão do LT                     │
│   • Determina maxUnavailable (padrão: 1 node, máximo: 100)          │
│                                                                      │
│ Fase 2: SCALE UP                                                     │
│   • Aumenta max e desired count do ASG                               │
│   • Lança novos nodes (nova AMI/versão) nas mesmas AZs               │
│   • Aguarda novos nodes ficarem Ready com labels EKS                 │
│   • Cordon + label "exclude-from-external-load-balancers" nos old    │
│   • Timeout: 15 min por node para bootar e entrar no cluster         │
│                                                                      │
│ Fase 3: UPGRADE (default strategy)                                   │
│   • Seleciona node antigo aleatoriamente (respeita maxUnavailable)   │
│   • Drena pods do node (eviction — respeita PDB)                    │
│   • Aguarda 60 segundos após cordon                                  │
│   • Envia terminação ao ASG                                          │
│   • Repete até todos os nodes usarem nova LT version                 │
│   • PodEvictionFailure → upgrade falha (se 15 min de timeout)       │
│                                                                      │
│ Fase 4: SCALE DOWN                                                   │
│   • Retorna max e desired count do ASG aos valores originais         │
│   • Se Cluster Autoscaler estiver escalando durante este passo,      │
│     o workflow sai imediatamente                                      │
└─────────────────────────────────────────────────────────────────────┘
```

[FATO] Há dois **update strategies** para a Fase 3:
- **Default** (recomendado): lança nodes novos primeiro, depois termina os velhos — capacidade total nunca cai abaixo do valor configurado
- **Minimal**: termina nodes velhos primeiro, depois lança novos — capacidade cai temporariamente; indicado para GPU (evitar pagar por dois sets de GPU simultaneamente)

[FATO] Durante um version update, o EKS **respeita Pod Disruption Budgets (PDB)**. Porém PDBs **não são respeitados** em operações de AZRebalance ou redução de desired count — essas ações tentam eviction por até 15 minutos e terminam o node independentemente.

---

## 4. kubectl drain — Zero-Downtime Node Maintenance

### 4.1 Diferença entre cordon e drain

```
kubectl cordon <node>
  → Marca o node como Unschedulable
  → Pods EXISTENTES continuam rodando
  → Nenhum pod NOVO é agendado no node
  → Usado quando você precisa de tempo para preparar o drain

kubectl drain <node>
  → Faz o cordon automaticamente
  → EVICTS todos os pods (exceto DaemonSets se --ignore-daemonsets)
  → Aguarda cada pod ser reagendado em outro node
  → Indica que o node está pronto para manutenção/desligamento
```

### 4.2 Flags essenciais do drain

[FATO] Flags obrigatórias para drain em clusters reais:

```bash
kubectl drain <node-name> \
  --ignore-daemonsets \          # obrigatório: DaemonSet pods não podem ser evicted
  --delete-emptydir-data \       # obrigatório: pods com emptyDir (local) seriam bloqueados
  --grace-period=60 \            # override do terminationGracePeriodSeconds (60s padrão)
  --timeout=300s \               # tempo máximo total para o drain completar
  --force                        # força remoção de pods sem controller (não gerenciados)
```

[FATO] Se um Pod tiver um **PDB que bloqueia a eviction** (ex.: `minAvailable=1` e só há 1 replica), o drain fica aguardando indefinidamente (ou até o `--timeout`). Solução: aumentar replicas temporariamente, ou ajustar o PDB.

### 4.3 PodDisruptionBudget — pré-requisito para drain seguro

```yaml
# PDB para garantir que sempre há ao menos 1 pod disponível durante manutenção
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: checkout-api-pdb
  namespace: production
spec:
  # minAvailable: mínimo de pods que devem estar Up durante disruption
  minAvailable: 1
  # OU maxUnavailable: máximo de pods que podem estar Down durante disruption
  # maxUnavailable: 1
  selector:
    matchLabels:
      app: checkout-api
```

[FATO] `minAvailable` e `maxUnavailable` podem ser números absolutos ou percentuais (ex.: `50%`). Com `minAvailable: 1` e 3 replicas, o drain pode evict até 2 pods por vez (2 maxUnavailable).

---

## 5. CDK Python — Node Groups, Fargate Profile, Taints e Labels

```python
from aws_cdk import (
    Stack,
    aws_eks as eks,
    aws_ec2 as ec2,
    aws_iam as iam,
)
from constructs import Construct


class EksNodeGroupsStack(Stack):
    def __init__(self, scope: Construct, construct_id: str,
                 cluster: eks.Cluster, node_role: iam.Role, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # ──────────────────────────────────────────────────────────────
        # Node group 1: Application — On-Demand, sem taint (workload principal)
        # ──────────────────────────────────────────────────────────────
        ng_app = cluster.add_nodegroup_capacity("NgApp",
            nodegroup_name="ng-app-ondemand",
            instance_types=[
                ec2.InstanceType("m5.xlarge"),
                ec2.InstanceType("m5a.xlarge"),
            ],
            min_size=3,
            desired_size=6,
            max_size=30,
            disk_size=100,
            node_role=node_role,
            capacity_type=eks.CapacityType.ON_DEMAND,
            labels={"tier": "app", "workload": "api"},
            # Sem taint — aceita qualquer pod que não tenha nodeSelector/affinity
            tags={"project": "checkout", "env": "prod"},
        )

        # ──────────────────────────────────────────────────────────────
        # Node group 2: Batch/Jobs — Spot, com taint NoSchedule
        # Apenas pods com toleration tier=batch:NoSchedule são agendados aqui
        # ──────────────────────────────────────────────────────────────
        ng_batch = cluster.add_nodegroup_capacity("NgBatch",
            nodegroup_name="ng-batch-spot",
            instance_types=[
                ec2.InstanceType("c5.xlarge"),
                ec2.InstanceType("c5a.xlarge"),
                ec2.InstanceType("c5n.xlarge"),
                ec2.InstanceType("c4.xlarge"),
                ec2.InstanceType("m5.xlarge"),   # diversificar pools Spot
                ec2.InstanceType("m5a.xlarge"),
            ],
            min_size=0,
            desired_size=2,
            max_size=50,
            disk_size=50,
            node_role=node_role,
            capacity_type=eks.CapacityType.SPOT,
            labels={"tier": "batch", "workload": "jobs"},
            taints=[
                eks.TaintSpec(
                    key="tier",
                    value="batch",
                    effect=eks.TaintEffect.NO_SCHEDULE,  # impede pods sem toleration
                )
            ],
        )

        # ──────────────────────────────────────────────────────────────
        # Node group 3: Infra — On-Demand, taint PreferNoSchedule
        # Monitoring, logging, cluster-autoscaler etc.
        # PreferNoSchedule: pods SEM toleration preferem outros nodes mas podem usar este
        # ──────────────────────────────────────────────────────────────
        ng_infra = cluster.add_nodegroup_capacity("NgInfra",
            nodegroup_name="ng-infra",
            instance_types=[ec2.InstanceType("m5.large")],
            min_size=2,
            desired_size=2,
            max_size=5,
            node_role=node_role,
            capacity_type=eks.CapacityType.ON_DEMAND,
            labels={"tier": "infra"},
            taints=[
                eks.TaintSpec(
                    key="tier",
                    value="infra",
                    effect=eks.TaintEffect.PREFER_NO_SCHEDULE,
                )
            ],
        )

        # ──────────────────────────────────────────────────────────────
        # Fargate Profile — namespace analytics e batch
        # ──────────────────────────────────────────────────────────────
        fargate_profile = cluster.add_fargate_profile("FargateAnalytics",
            fargate_profile_name="fp-analytics",
            selectors=[
                # Analytics: somente pods com label fargate=true
                eks.Selector(
                    namespace="analytics",
                    labels={"fargate": "true"},
                ),
                # Batch jobs: qualquer pod no namespace batch
                eks.Selector(
                    namespace="batch-fargate",
                ),
            ],
            # Fargate requer subnets privadas
            subnet_selection=ec2.SubnetSelection(
                subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS,
            ),
        )


class EksFargateWorkloadStack(Stack):
    """
    Exemplos de Kubernetes manifests aplicados via CDK (eks.KubernetesManifest).
    """
    def __init__(self, scope: Construct, construct_id: str,
                 cluster: eks.Cluster, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # ──────────────────────────────────────────────────────────────
        # PodDisruptionBudget para o workload de API
        # ──────────────────────────────────────────────────────────────
        cluster.add_manifest("CheckoutApiPDB", {
            "apiVersion": "policy/v1",
            "kind": "PodDisruptionBudget",
            "metadata": {"name": "checkout-api-pdb", "namespace": "production"},
            "spec": {
                "minAvailable": 1,
                "selector": {"matchLabels": {"app": "checkout-api"}},
            },
        })

        # ──────────────────────────────────────────────────────────────
        # Deployment de API — nodeSelector para ng-app, sem toleration de batch
        # ──────────────────────────────────────────────────────────────
        cluster.add_manifest("CheckoutApiDeployment", {
            "apiVersion": "apps/v1",
            "kind": "Deployment",
            "metadata": {"name": "checkout-api", "namespace": "production"},
            "spec": {
                "replicas": 3,
                "selector": {"matchLabels": {"app": "checkout-api"}},
                "template": {
                    "metadata": {"labels": {"app": "checkout-api"}},
                    "spec": {
                        # Garante agendamento apenas em nodes com tier=app
                        "nodeSelector": {"tier": "app"},
                        # topologySpreadConstraints: distribui réplicas entre AZs
                        "topologySpreadConstraints": [{
                            "maxSkew": 1,
                            "topologyKey": "topology.kubernetes.io/zone",
                            "whenUnsatisfiable": "DoNotSchedule",
                            "labelSelector": {"matchLabels": {"app": "checkout-api"}},
                        }],
                        "containers": [{
                            "name": "api",
                            "image": "my-ecr.dkr.ecr.us-east-1.amazonaws.com/checkout-api:latest",
                            "resources": {
                                "requests": {"cpu": "500m", "memory": "512Mi"},
                                "limits":   {"cpu": "2000m", "memory": "2Gi"},
                            },
                            # terminationGracePeriodSeconds padrão é 30s
                        }],
                        "terminationGracePeriodSeconds": 60,
                    },
                },
            },
        })

        # ──────────────────────────────────────────────────────────────
        # Job batch no node group Spot — com toleration de tier=batch
        # ──────────────────────────────────────────────────────────────
        cluster.add_manifest("BatchJob", {
            "apiVersion": "batch/v1",
            "kind": "Job",
            "metadata": {"name": "report-generator", "namespace": "batch"},
            "spec": {
                "ttlSecondsAfterFinished": 3600,   # limpa job 1h após completar
                "template": {
                    "spec": {
                        "restartPolicy": "Never",
                        "nodeSelector": {"tier": "batch"},
                        "tolerations": [{
                            "key": "tier",
                            "value": "batch",
                            "effect": "NoSchedule",
                            "operator": "Equal",
                        }],
                        "containers": [{
                            "name": "report",
                            "image": "my-ecr.dkr.ecr.us-east-1.amazonaws.com/report-gen:v1",
                            "resources": {
                                "requests": {"cpu": "2000m", "memory": "4Gi"},
                            },
                        }],
                    },
                },
            },
        })

        # ──────────────────────────────────────────────────────────────
        # Pod no Fargate — namespace analytics com label fargate=true
        # IMDS indisponível: sem AWS_METADATA_SERVICE_TIMEOUT hardcoded
        # Credenciais via IRSA (serviceAccountName + annotation)
        # ──────────────────────────────────────────────────────────────
        cluster.add_manifest("AnalyticsPod", {
            "apiVersion": "apps/v1",
            "kind": "Deployment",
            "metadata": {"name": "analytics-processor", "namespace": "analytics"},
            "spec": {
                "replicas": 2,
                "selector": {"matchLabels": {"app": "analytics-processor"}},
                "template": {
                    "metadata": {
                        "labels": {
                            "app": "analytics-processor",
                            "fargate": "true",    # match Fargate profile selector
                        },
                    },
                    "spec": {
                        # serviceAccountName com annotation IRSA para credenciais IAM
                        "serviceAccountName": "analytics-sa",
                        "containers": [{
                            "name": "processor",
                            "image": "my-ecr.dkr.ecr.us-east-1.amazonaws.com/analytics:v2",
                            "resources": {
                                # Fargate arredonda para combinação suportada
                                "requests": {"cpu": "1000m", "memory": "2Gi"},
                                "limits":   {"cpu": "1000m", "memory": "2Gi"},
                            },
                            "env": [
                                # AWS_REGION hard-coded — IMDS indisponível no Fargate
                                {"name": "AWS_REGION", "value": "us-east-1"},
                                {"name": "AWS_DEFAULT_REGION", "value": "us-east-1"},
                            ],
                        }],
                        # Fargate ignora nodeSelector — o pod é agendado via profile
                    },
                },
            },
        })
```

---

## 6. Python — Script de upgrade orchestration

```python
"""
Script de upgrade do cluster EKS.
Valida pré-condições e executa upgrade em ordem correta.
"""
import boto3
import subprocess
import time
import sys
from dataclasses import dataclass

eks = boto3.client("eks")


@dataclass
class ClusterState:
    name: str
    current_version: str
    target_version: str
    status: str
    node_groups: list[dict]
    addons: list[dict]


def get_cluster_state(cluster_name: str, target_version: str) -> ClusterState:
    cluster = eks.describe_cluster(name=cluster_name)["cluster"]
    ng_resp = eks.list_nodegroups(clusterName=cluster_name)
    node_groups = []
    for ng_name in ng_resp["nodegroups"]:
        ng = eks.describe_nodegroup(clusterName=cluster_name, nodegroupName=ng_name)["nodegroup"]
        node_groups.append({
            "name": ng_name,
            "version": ng["version"],
            "status": ng["status"],
        })
    addon_resp = eks.list_addons(clusterName=cluster_name)
    addons = []
    for addon_name in addon_resp["addons"]:
        addon = eks.describe_addon(clusterName=cluster_name, addonName=addon_name)["addon"]
        addons.append({"name": addon_name, "version": addon["addonVersion"], "status": addon["status"]})

    return ClusterState(
        name=cluster_name,
        current_version=cluster["version"],
        target_version=target_version,
        status=cluster["status"],
        node_groups=node_groups,
        addons=addons,
    )


def validate_upgrade_prerequisites(state: ClusterState) -> list[str]:
    """Valida pré-condições para o upgrade. Retorna lista de erros."""
    errors = []

    # 1. Verificar status do cluster
    if state.status != "ACTIVE":
        errors.append(f"Cluster não está ACTIVE: {state.status}")

    # 2. Verificar version skew (apenas 1 minor version)
    curr_parts = [int(x) for x in state.current_version.split(".")]
    tgt_parts  = [int(x) for x in state.target_version.split(".")]
    if tgt_parts[1] - curr_parts[1] != 1:
        errors.append(
            f"Upgrade deve ser 1 minor version por vez: {state.current_version} → {state.target_version} "
            f"(diferença: {tgt_parts[1] - curr_parts[1]} minor versions)"
        )

    # 3. Verificar node groups — todos devem estar na versão atual do control plane
    for ng in state.node_groups:
        if ng["version"] != state.current_version:
            errors.append(
                f"Node group {ng['name']} está na versão {ng['version']}, "
                f"mas control plane está em {state.current_version}. "
                "Atualize o node group antes de fazer upgrade do control plane."
            )
        if ng["status"] != "ACTIVE":
            errors.append(f"Node group {ng['name']} não está ACTIVE: {ng['status']}")

    return errors


def wait_for_cluster_active(cluster_name: str, timeout_min: int = 30) -> bool:
    """Aguarda cluster ficar ACTIVE. Retorna True se sucesso."""
    deadline = time.time() + timeout_min * 60
    while time.time() < deadline:
        status = eks.describe_cluster(name=cluster_name)["cluster"]["status"]
        print(f"  Cluster status: {status}")
        if status == "ACTIVE":
            return True
        if status == "FAILED":
            return False
        time.sleep(30)
    return False


def upgrade_control_plane(state: ClusterState) -> str:
    """Inicia upgrade do control plane. Retorna update_id."""
    print(f"\n[Step 2] Upgrading control plane {state.current_version} → {state.target_version}...")
    response = eks.update_cluster_version(
        name=state.name,
        kubernetesVersion=state.target_version,
    )
    update_id = response["update"]["id"]
    print(f"  Update iniciado: {update_id}")
    return update_id


def wait_for_cluster_update(cluster_name: str, update_id: str, timeout_min: int = 30) -> bool:
    deadline = time.time() + timeout_min * 60
    while time.time() < deadline:
        resp = eks.describe_update(name=cluster_name, updateId=update_id)
        status = resp["update"]["status"]
        print(f"  Update status: {status}")
        if status == "Successful":
            return True
        if status in ("Cancelled", "Failed"):
            print(f"  Update falhou: {resp['update'].get('errors', [])}")
            return False
        time.sleep(30)
    return False


def upgrade_addon(cluster_name: str, addon_name: str) -> bool:
    """Atualiza um add-on para a versão mais recente compatível com o cluster."""
    try:
        # Obter versão atual do cluster
        cluster_version = eks.describe_cluster(name=cluster_name)["cluster"]["version"]

        # Listar versões compatíveis e escolher a mais recente
        versions_resp = eks.describe_addon_versions(
            kubernetesVersion=cluster_version,
            addonName=addon_name,
        )
        latest = versions_resp["addons"][0]["addonVersions"][0]["addonVersion"]

        print(f"  Atualizando add-on {addon_name} → {latest}")
        eks.update_addon(
            clusterName=cluster_name,
            addonName=addon_name,
            addonVersion=latest,
            resolveConflicts="OVERWRITE",
        )
        # Aguarda add-on ficar ACTIVE
        for _ in range(20):
            status = eks.describe_addon(clusterName=cluster_name, addonName=addon_name)
            if status["addon"]["status"] == "ACTIVE":
                return True
            time.sleep(15)
        return False
    except Exception as e:
        print(f"  Erro ao atualizar add-on {addon_name}: {e}")
        return False


def upgrade_nodegroup(cluster_name: str, ng_name: str, target_version: str) -> bool:
    """Inicia rolling update do node group."""
    print(f"\n  Upgrading node group {ng_name} → {target_version}...")
    try:
        resp = eks.update_nodegroup_version(
            clusterName=cluster_name,
            nodegroupName=ng_name,
            version=target_version,
        )
        update_id = resp["update"]["id"]
        # Aguarda conclusão (pode levar 30-60 min para node groups grandes)
        deadline = time.time() + 90 * 60   # 90 min timeout
        while time.time() < deadline:
            status = eks.describe_update(
                name=cluster_name,
                updateId=update_id,
                nodegroupName=ng_name,
            )["update"]["status"]
            print(f"    Node group {ng_name} update: {status}")
            if status == "Successful":
                return True
            if status in ("Cancelled", "Failed"):
                return False
            time.sleep(30)
        return False
    except Exception as e:
        print(f"  Erro ao atualizar node group {ng_name}: {e}")
        return False


def run_cluster_upgrade(cluster_name: str, target_version: str) -> None:
    print(f"=== EKS Cluster Upgrade: {cluster_name} → {target_version} ===\n")

    # Step 1: Validar pré-condições
    print("[Step 1] Validando pré-condições...")
    state = get_cluster_state(cluster_name, target_version)
    errors = validate_upgrade_prerequisites(state)
    if errors:
        print("  ERROS ENCONTRADOS — upgrade abortado:")
        for err in errors:
            print(f"  ✗ {err}")
        sys.exit(1)
    print("  ✓ Pré-condições OK")

    # Step 2: Upgrade control plane
    update_id = upgrade_control_plane(state)
    if not wait_for_cluster_update(cluster_name, update_id):
        print("  ✗ Upgrade do control plane falhou")
        sys.exit(1)
    print("  ✓ Control plane atualizado")

    # Step 3: Upgrade add-ons
    print("\n[Step 3] Atualizando add-ons...")
    for addon_name in ["vpc-cni", "coredns", "kube-proxy", "aws-ebs-csi-driver"]:
        if not upgrade_addon(cluster_name, addon_name):
            print(f"  ✗ Falha ao atualizar add-on {addon_name}")

    # Step 4: Upgrade node groups
    print("\n[Step 4] Atualizando node groups...")
    state = get_cluster_state(cluster_name, target_version)  # reler estado atual
    for ng in state.node_groups:
        if ng["version"] != target_version:
            if not upgrade_nodegroup(cluster_name, ng["name"], target_version):
                print(f"  ✗ Falha ao atualizar node group {ng['name']}")
            else:
                print(f"  ✓ Node group {ng['name']} atualizado")

    print("\n=== Upgrade completo ===")
    print(f"Próximo passo: kubectl rollout restart (Fargate pods não são auto-atualizados)")
```

---

## 7. CLI — Exemplos Essenciais

```bash
# ─────────────────────────────────────────────────────────────
# MANAGED NODE GROUPS — labels e taints
# ─────────────────────────────────────────────────────────────

# Adicionar/atualizar labels e taints no node group (sem recriar nodes)
aws eks update-nodegroup-config \
  --cluster-name checkout-prod \
  --nodegroup-name ng-batch-spot \
  --labels 'addOrUpdateLabels={tier=batch,updated-at=2026-06-13}' \
  --taints 'addOrUpdateTaints=[{key=tier,value=batch,effect=NO_SCHEDULE}]'

# Remover label de um node group
aws eks update-nodegroup-config \
  --cluster-name checkout-prod \
  --nodegroup-name ng-batch-spot \
  --labels 'removeLabels=[updated-at]'

# Remover taint de um node group
aws eks update-nodegroup-config \
  --cluster-name checkout-prod \
  --nodegroup-name ng-batch-spot \
  --taints 'removeTaints=[{key=tier,effect=NO_SCHEDULE}]'

# Listar todos os node groups com versão e status
aws eks list-nodegroups --cluster-name checkout-prod
aws eks describe-nodegroup \
  --cluster-name checkout-prod \
  --nodegroup-name ng-app-ondemand \
  --query 'nodegroup.{Name:nodegroupName,Version:version,Status:status,Capacity:capacityType,Labels:labels,Taints:taints}'

# ─────────────────────────────────────────────────────────────
# FARGATE PROFILES
# ─────────────────────────────────────────────────────────────

# Criar Fargate profile via CLI
aws eks create-fargate-profile \
  --cluster-name checkout-prod \
  --fargate-profile-name fp-analytics \
  --pod-execution-role-arn arn:aws:iam::123456789012:role/EKSFargatePodExecutionRole \
  --subnets subnet-0111aaaa subnet-0222bbbb subnet-0333cccc \
  --selectors 'namespace=analytics,labels={fargate=true}' \
              'namespace=batch-fargate'

# Listar Fargate profiles
aws eks list-fargate-profiles --cluster-name checkout-prod

# Verificar status e selectors de um profile
aws eks describe-fargate-profile \
  --cluster-name checkout-prod \
  --fargate-profile-name fp-analytics \
  --query 'fargateProfile.{Status:status,Selectors:selectors}'

# ─────────────────────────────────────────────────────────────
# CLUSTER UPGRADE
# ─────────────────────────────────────────────────────────────

# Step 0: Verificar upgrade insights (API deprecated, issues de compatibilidade)
aws eks list-insights \
  --cluster-name checkout-prod \
  --filter '{"categories":["UPGRADE_READINESS"]}' \
  --query 'insights[*].{Name:name,Status:insightStatus.status,Recommendation:recommendation}'

# Step 1: Verificar versão atual e nodes
kubectl version
kubectl get nodes -o wide

# Step 2: Upgrade control plane (aguardar ~15 min)
aws eks update-cluster-version \
  --name checkout-prod \
  --kubernetes-version 1.33 \
  --region us-east-1

# Monitorar progresso
aws eks describe-cluster --name checkout-prod \
  --query 'cluster.{Version:version,Status:status,PlatformVersion:platformVersion}'

# Step 3: Upgrade add-ons
for addon in vpc-cni coredns kube-proxy aws-ebs-csi-driver; do
  echo "Upgrading add-on: $addon"
  LATEST=$(aws eks describe-addon-versions \
    --kubernetes-version 1.33 \
    --addon-name "$addon" \
    --query 'addons[0].addonVersions[0].addonVersion' \
    --output text)
  aws eks update-addon \
    --cluster-name checkout-prod \
    --addon-name "$addon" \
    --addon-version "$LATEST" \
    --resolve-conflicts OVERWRITE
done

# Step 4: Upgrade node groups (um por vez, aguardar conclusão)
aws eks update-nodegroup-version \
  --cluster-name checkout-prod \
  --nodegroup-name ng-app-ondemand \
  --kubernetes-version 1.33   # ou sem --kubernetes-version para só atualizar AMI

# Monitorar node group update
aws eks describe-update \
  --name checkout-prod \
  --nodegroup-name ng-app-ondemand \
  --update-id <update-id> \
  --query 'update.{Status:status,Type:type,Errors:errors}'

# Step 5: Redeployar pods Fargate (não são auto-atualizados)
kubectl rollout restart deployment/analytics-processor -n analytics

# ─────────────────────────────────────────────────────────────
# kubectl drain — manutenção de node
# ─────────────────────────────────────────────────────────────

# Listar nodes e identificar o que será drenado
kubectl get nodes -o wide

# Cordon (impede novos pods sem evictar existentes)
kubectl cordon ip-10-0-2-10.us-east-1.compute.internal

# Verificar pods no node
kubectl get pods -A -o wide --field-selector spec.nodeName=ip-10-0-2-10.us-east-1.compute.internal

# Drain (cordon + evict)
kubectl drain ip-10-0-2-10.us-east-1.compute.internal \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=60 \
  --timeout=300s

# Verificar que todos os pods foram movidos
kubectl get pods -A -o wide | grep ip-10-0-2-10

# Após manutenção: uncordon para voltar a aceitar pods
kubectl uncordon ip-10-0-2-10.us-east-1.compute.internal

# Verificar PDBs que podem bloquear drain
kubectl get pdb -A
kubectl describe pdb checkout-api-pdb -n production
```

---

## 8. Armadilhas (Pitfalls)

[FATO] **Fargate ignora nodeSelector** — pods agendados no Fargate não respeitam `nodeSelector` definido pelo usuário. O critério de agendamento é exclusivamente o Fargate Profile (namespace + labels). Se um pod não tiver match em nenhum profile, fica Pending.

[FATO] **DaemonSets existentes em clusters com Fargate** causam Pending pods: o scheduler do Fargate tenta criar um DaemonSet pod no "Fargate node", mas DaemonSets não são suportados. Solução: adicionar `nodeAffinity` ao DaemonSet para rodar apenas em EC2 nodes (via label `eks.amazonaws.com/compute-type: ec2`).

[FATO] **Upgrade do control plane pode falhar silenciosamente se subnets estiverem cheias**: o erro `InsufficientFreeAddressesInSubnet` aparece no describe-update mas o cluster continua na versão anterior (rollback automático). Verifique IPs disponíveis nas subnets do cluster antes do upgrade.

[FATO] **Node groups Spot com `maxUnavailable=1` e muitas AZs podem lançar muitos nodes temporariamente**: o scale up lança até `2 × número de AZs` nodes (ex.: 3 AZs = 6 nodes extras antes de drenar os velhos). Isso pode acionar limites de EC2 instance quota.

[CONSENSO] **PDB com `minAvailable=N` onde N = número de replicas bloqueia drain indefinidamente**: se o deployment tem 2 replicas e o PDB exige `minAvailable=2`, nenhum pod pode ser evictado. Sempre configure PDB com folga (`minAvailable = total_replicas - 1` ou `maxUnavailable >= 1`).

[FATO] **Fargate jobs precisam de `ttlSecondsAfterFinished`**: pods de Jobs completados no Fargate continuam gerando cobrança (CPU/memória do pod) após a conclusão. Sempre configure TTL para evitar custos de pods zumbis.

---

## Exercício de Reflexão

Você precisa redesenhar a topologia de compute de um cluster EKS com 4 tipos de workload:

1. **API Gateway** (stateless, alto tráfego, SLA 99.9%) — 10 replicas, sem tolerância a interruption
2. **ML Inference** (stateless, CPU-intensivo, tolerante a 10% de interrupções) — 50 replicas
3. **Batch ETL** (jobs curtos, 5-30 min, altamente tolerante a interrupções) — 0–200 pods variável
4. **Monitoring Stack** (Prometheus, Grafana — stateful, DaemonSets necessários) — 3 replicas

**Responda:**

1. Quantos managed node groups são necessários? Qual capacity type (OD/Spot) para cada um? Quais labels e taints?
2. O ML Inference deve ir para Fargate ou EC2 node group? Por quê? (considere: DaemonSets para métricas, volume de pods, cost)
3. O Monitoring Stack pode ir para Fargate? Por quê?
4. Como configurar PDBs para garantir que o `kubectl drain` funcione sem bloquear por mais de 5 minutos para cada workload?
5. Durante o cluster upgrade 1.32→1.33, em que ordem exata devem ser atualizados os 4 node groups? Existe alguma dependência de ordem?

---

## Referências

- [FATO] [Simplify node lifecycle with managed node groups](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html) — docs.aws.amazon.com
- [FATO] [Understand each phase of node updates](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-update-behavior.html) — docs.aws.amazon.com
- [FATO] [Simplify compute management with AWS Fargate](https://docs.aws.amazon.com/eks/latest/userguide/fargate.html) — docs.aws.amazon.com
- [FATO] [Define which Pods use AWS Fargate when launched](https://docs.aws.amazon.com/eks/latest/userguide/fargate-profile.html) — docs.aws.amazon.com
- [FATO] [Update existing cluster to new Kubernetes version](https://docs.aws.amazon.com/eks/latest/userguide/update-cluster.html) — docs.aws.amazon.com
- [FATO] [Recipe: Prevent pods from being scheduled on specific nodes (taints)](https://docs.aws.amazon.com/eks/latest/userguide/node-taints-managed-node-groups.html) — docs.aws.amazon.com
