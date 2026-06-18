# Sessão 049 — EKS: Add-ons Gerenciados — VPC CNI, CoreDNS e EBS CSI Driver

**Duração estimada:** 60 min  
**Pré-requisito:** session-048 (IRSA e Pod Identity)

---

## Objetivos da sessão

- Entender o modelo de add-ons gerenciados do EKS: categorias, conflitos, field management
- Instalar, configurar e atualizar add-ons via CLI usando `--configuration-values` e `--resolve-conflicts`
- Configurar o VPC CNI para Prefix Delegation e calcular o ganho em densidade de pods
- Instalar o EBS CSI Driver com IAM correto (IRSA ou Pod Identity) e provisionar PersistentVolumes dinâmicos

---

## 1. Modelo de Add-ons Gerenciados

### 1.1 Categorias de add-ons

[FATO] Existem três categorias de add-ons no EKS, com níveis distintos de suporte:

```
╔══════════════════════╦════════════════════════════════╦══════════════╗
║ Categoria            ║ Exemplos                       ║ Suporte      ║
╠══════════════════════╬════════════════════════════════╬══════════════╣
║ AWS Add-ons          ║ VPC CNI, CoreDNS, kube-proxy,  ║ Full AWS     ║
║                      ║ EBS CSI, Pod Identity Agent,   ║ Support      ║
║                      ║ EFS CSI, S3 CSI                ║              ║
╠══════════════════════╬════════════════════════════════╬══════════════╣
║ AWS Marketplace      ║ Splunk, Datadog, Dynatrace,    ║ Partner      ║
║ Add-ons              ║ Tetrate, Calico                ║ Support      ║
╠══════════════════════╬════════════════════════════════╬══════════════╣
║ Community Add-ons    ║ Metrics Server, cert-manager,  ║ Community    ║
║                      ║ external-dns, Argo CD          ║ (AWS valida  ║
║                      ║                                ║  compat K8s) ║
╚══════════════════════╩════════════════════════════════╩══════════════╝
```

[FATO] Os três add-ons **auto-instalados** pelo EKS em todo cluster novo:
- `amazon-vpc-cni` (VPC CNI) — gerencia ENIs e IPs dos pods
- `coredns` — DNS interno do cluster
- `kube-proxy` — roteamento de rede de Services

Quando criados via Console, esses três são instalados como add-ons **gerenciados**. Quando criados via `eksctl` sem `config` file, são instalados como **self-managed** (sem gerenciamento EKS).

### 1.2 Kubernetes Field Management (Server-Side Apply)

[FATO] O EKS usa **Kubernetes Server-Side Apply** (SSA) para gerenciar os campos dos add-ons. Isso significa:

```
┌──────────────────────────────────────────────────────────────┐
│ Campo gerenciado pelo EKS (field manager: eks)               │
│ → EKS sobrescreve se você mudar via kubectl apply            │
│                                                              │
│ Campo NÃO gerenciado pelo EKS                                │
│ → Você pode mudar via kubectl; EKS não sobrescreve           │
└──────────────────────────────────────────────────────────────┘
```

Para ver quais campos um add-on pode configurar via `--configuration-values`:

```bash
aws eks describe-addon-configuration \
  --addon-name amazon-vpc-cni \
  --addon-version v1.21.1-eksbuild.8 \
  --query 'schema' \
  --output text | python3 -m json.tool
```

[FATO] O `eks:addon-manager` tem um `ClusterRoleBinding` chamado `eks:addon-cluster-admin` que vincula o role `cluster-admin` ao identity `eks:addon-manager`. Se esse `ClusterRoleBinding` for removido, o EKS perde a capacidade de gerenciar add-ons.

### 1.3 Resolve Conflicts

[FATO] O parâmetro `--resolve-conflicts` controla o que acontece quando o add-on EKS entra em conflito com configurações existentes:

```
OVERWRITE  → EKS sobrescreve com valores padrão (usar em clusters novos)
PRESERVE   → EKS mantém valores existentes (usar em atualizações com config customizada)
NONE       → EKS não muda nada; criação pode falhar se houver conflito
```

[FATO] Ao **atualizar** um add-on com `--resolve-conflicts PRESERVE`, o EKS mantém os campos que você customizou e só atualiza campos que nunca foram alterados.

---

## 2. VPC CNI — Prefix Delegation

### 2.1 Recap: Secondary IP Mode vs Prefix Delegation

[FATO] Em Secondary IP Mode (padrão), cada slot de IP secundário em uma ENI recebe **1 endereço IP**, que é atribuído a um pod. A fórmula de max pods é:

```
max_pods = (num_ENIs × (IPs_por_ENI - 1)) + 2

Exemplo m5.xlarge: 4 ENIs × (15 - 1) + 2 = 58 pods
```

[FATO] Em **Prefix Delegation Mode**, cada slot recebe um prefixo **/28 (16 IPs)** ao invés de 1 IP. Isso multiplica a densidade por até 16×:

```
max_pods = (num_ENIs × ((IPs_por_ENI - 1) × 16)) + 2

Exemplo m5.xlarge: 4 × (14 × 16) + 2 = 898 pods (limitado pelo kubelet)
```

[FATO] Prefix Delegation está disponível apenas em instâncias **Nitro** e requer VPC CNI ≥ v1.9.0.

[FATO] A migração de Secondary IP para Prefix Delegation é **irreversível por nó**. A recomendação AWS é criar novos node groups em vez de fazer rolling replace dos existentes. Um nó com mistura de IPs e prefixos pode reportar capacidade inconsistente.

### 2.2 Variáveis de controle do warm pool

[FATO] Com Prefix Delegation habilitado, as variáveis de controle relevantes são:

```
ENABLE_PREFIX_DELEGATION=true   → Habilita prefix delegation (obrigatório)
WARM_PREFIX_TARGET=1            → Mantém 1 prefixo /28 "quente" por ENI (padrão)
WARM_IP_TARGET=N                → (alternativa) mantém N IPs individuais quentes
MINIMUM_IP_TARGET=N             → Garante mínimo de N IPs disponíveis no pool
```

[FATO] `WARM_PREFIX_TARGET` e `WARM_IP_TARGET` são **mutuamente exclusivos** como estratégia principal. Use `WARM_PREFIX_TARGET` para burst rápido; use `WARM_IP_TARGET` para controle fino de custo (menos IPs ociosos).

### 2.3 Limite de pods por nó — ajuste necessário

[FATO] O padrão do kubelet é 110 pods por nó. Com Prefix Delegation, você precisa aumentar esse limite explicitamente via `maxPodsPerNode` no launch template ou no campo `--max-pods` do kubelet. O script `max-pods-calculator.sh` da AWS calcula o valor correto por tipo de instância.

---

## 3. CoreDNS — Configuração e Escala

[FATO] O CoreDNS roda por padrão com **2 réplicas** no namespace `kube-system`. O add-on gerenciado expõe configuração de recursos via `--configuration-values`:

```json
{
  "replicaCount": 2,
  "resources": {
    "limits":   { "cpu": "100m", "memory": "170Mi" },
    "requests": { "cpu": "100m", "memory": "70Mi"  }
  },
  "corefile": ""
}
```

[FATO] Para ambientes de produção com alto volume de resolução DNS, o padrão de 2 réplicas pode ser insuficiente. Escalar para 3–5 réplicas é prática comum em clusters com centenas de pods. O CoreDNS também pode ser escalado com HPA baseado em memória ou CPU.

[FATO] Se você customizar o `Corefile` diretamente via `kubectl edit configmap -n kube-system coredns`, esse campo **não** é gerenciado pelo EKS (SSA), então o EKS não sobrescreve. Para incluir no versionamento, usar `--configuration-values` com a chave `corefile`.

---

## 4. EBS CSI Driver

### 4.1 O que o EBS CSI Driver faz

[FATO] O `aws-ebs-csi-driver` é o CSI (Container Storage Interface) plugin que gerencia o ciclo de vida de volumes Amazon EBS como `PersistentVolumes` Kubernetes. Ele permite:
- Provisionar volumes EBS dinamicamente ao criar PVCs
- Redimensionar volumes online (`allowVolumeExpansion: true`)
- Tirar snapshots EBS via `VolumeSnapshot`
- Criptografar volumes com KMS customer-managed keys

[FATO] Componentes do driver:
- **Controller** (Deployment, 2 réplicas): provisiona, anexa, desanexa volumes; pode rodar em Fargate
- **Node** (DaemonSet): monta/desmonta volumes no nó; roda apenas em **EC2** (não Fargate)

[FATO] EBS não é suportado em Fargate pods (somente EFS é suportado em Fargate). Pods Fargate não podem montar PVCs EBS.

### 4.2 IAM — Três políticas disponíveis

[FATO] O EBS CSI Driver requer IAM para chamar EC2 APIs (CreateVolume, AttachVolume etc.). Há três opções de política gerenciada:

```
AmazonEBSCSIDriverPolicyV2          → Restrição por tag (recomendado)
                                       Permite criar volumes somente com
                                       tag kubernetes.io/cluster/<name>

AmazonEBSCSIDriverEKSClusterScopedPolicy → Restrição por cluster ARN
                                           Mais restritivo para multi-cluster

AmazonEBSCSIDriverPolicy            → Sem restrição de scope (legado)
                                       Evitar em novos deployments
```

[FATO] A política `AmazonEBSCSIDriverPolicyV2` requer que o `StorageClass` inclua a tag `kubernetes.io/cluster/<cluster-name>` nos volumes criados (o driver adiciona automaticamente quando o add-on está instalado corretamente).

### 4.3 StorageClass padrão

[FATO] O `aws-ebs-csi-driver` cria uma `StorageClass` padrão chamada `gp2` (legado) ou `gp3` (recomendado). Para usar `gp3` como padrão:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer    # CRÍTICO: evita cross-AZ binding
reclaimPolicy: Delete
allowVolumeExpansion: true
parameters:
  type: gp3
  encrypted: "true"
  # kmsKeyId: arn:aws:kms:...  # opcional: CMK
```

[FATO] `volumeBindingMode: WaitForFirstConsumer` é crítico: adia o provisionamento do volume EBS até um pod ser agendado em um nó específico. Isso garante que o volume seja criado na **mesma AZ do nó**. Com `Immediate`, o volume pode ser criado em uma AZ diferente do pod, causando falha de mount.

### 4.4 Snapshot Controller

[FATO] O suporte a `VolumeSnapshot` requer instalação separada do **CSI Snapshot Controller** — ele **não** vem incluído no `aws-ebs-csi-driver`. O snapshot controller é um CRD + controller:
- `VolumeSnapshotClass`
- `VolumeSnapshot`
- `VolumeSnapshotContent`

O add-on `aws-ebs-csi-driver` inclui um sidecar `csi-snapshotter` que se comunica com o snapshot controller, mas o controller em si deve ser instalado via community add-on ou Helm.

---

## 5. CDK Python — Add-ons e EBS CSI Driver

```python
from aws_cdk import (
    Stack,
    aws_eks as eks,
    aws_iam as iam,
    aws_kms as kms,
)
from constructs import Construct


class EksAddonsStack(Stack):
    """
    Gerencia add-ons EKS: VPC CNI (Prefix Delegation), CoreDNS e EBS CSI Driver.
    """
    def __init__(self, scope: Construct, construct_id: str,
                 cluster: eks.Cluster, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # ──────────────────────────────────────────────────────────────
        # VPC CNI — habilitar Prefix Delegation via configuration-values
        # Requer que o add-on já exista; usa CfnAddon (L1) para
        # acessar configuration_values
        # ──────────────────────────────────────────────────────────────
        vpc_cni_config = {
            "env": {
                "ENABLE_PREFIX_DELEGATION": "true",
                "WARM_PREFIX_TARGET":       "1",
                # Para controle fino de warm pool (alternativa ao WARM_PREFIX_TARGET):
                # "WARM_IP_TARGET":    "16",
                # "MINIMUM_IP_TARGET": "32",
                "AWS_VPC_K8S_CNI_LOGLEVEL": "WARN",   # reduz ruído de logs
            },
            # Resource limits para o DaemonSet aws-node
            "resources": {
                "requests": {"cpu": "25m"},
                "limits":   {"cpu": "100m"},
            },
        }

        import json
        vpc_cni_addon = eks.CfnAddon(self, "VpcCniAddon",
            cluster_name=cluster.cluster_name,
            addon_name="amazon-vpc-cni",
            # Não fixar versão → EKS usa a versão padrão para a versão K8s do cluster
            # addon_version="v1.21.1-eksbuild.8",
            resolve_conflicts_on_update="PRESERVE",   # manter customizações
            configuration_values=json.dumps(vpc_cni_config),
        )

        # ──────────────────────────────────────────────────────────────
        # CoreDNS — 3 réplicas em produção, recursos ajustados
        # ──────────────────────────────────────────────────────────────
        coredns_config = {
            "replicaCount": 3,
            "resources": {
                "limits":   {"cpu": "200m", "memory": "256Mi"},
                "requests": {"cpu": "100m", "memory": "70Mi"},
            },
            # podDisruptionBudget garante mínimo de 1 réplica durante drain
            "podDisruptionBudget": {
                "enabled": True,
                "minAvailable": 1,
            },
        }

        coredns_addon = eks.CfnAddon(self, "CoreDnsAddon",
            cluster_name=cluster.cluster_name,
            addon_name="coredns",
            resolve_conflicts_on_update="PRESERVE",
            configuration_values=json.dumps(coredns_config),
        )

        # ──────────────────────────────────────────────────────────────
        # EBS CSI Driver — IAM via Pod Identity (preferido) ou IRSA
        # ──────────────────────────────────────────────────────────────

        # Opção A: Pod Identity (recomendado para EKS EC2 nodes)
        ebs_csi_role_pod_id = iam.Role(self, "EbsCsiRolePodId",
            role_name="AmazonEKS_EBS_CSI_DriverRole",
            description="EBS CSI Driver role via Pod Identity",
            assumed_by=iam.ServicePrincipal("pods.eks.amazonaws.com"),
        )
        # TagSession obrigatório para Pod Identity
        ebs_csi_role_pod_id.assume_role_policy.add_statements(
            iam.PolicyStatement(
                effect=iam.Effect.ALLOW,
                principals=[iam.ServicePrincipal("pods.eks.amazonaws.com")],
                actions=["sts:AssumeRole", "sts:TagSession"],
            )
        )
        ebs_csi_role_pod_id.add_managed_policy(
            iam.ManagedPolicy.from_aws_managed_policy_name(
                "service-role/AmazonEBSCSIDriverPolicyV2"
            )
        )

        # KMS key para criptografia de volumes (opcional)
        ebs_kms_key = kms.Key(self, "EbsKmsKey",
            description="CMK para volumes EBS do cluster EKS",
            enable_key_rotation=True,
        )
        # Permissões KMS adicionais ao role do driver
        ebs_kms_key.add_to_resource_policy(iam.PolicyStatement(
            principals=[iam.ArnPrincipal(ebs_csi_role_pod_id.role_arn)],
            actions=[
                "kms:CreateGrant",
                "kms:ListGrants",
                "kms:RevokeGrant",
                "kms:Encrypt",
                "kms:Decrypt",
                "kms:ReEncrypt*",
                "kms:GenerateDataKey*",
                "kms:DescribeKey",
            ],
            resources=["*"],
            conditions={"Bool": {"kms:GrantIsForAWSResource": "true"}},
        ))

        # EBS CSI add-on com Pod Identity inline (pod-identity-associations)
        ebs_csi_addon = eks.CfnAddon(self, "EbsCsiAddon",
            cluster_name=cluster.cluster_name,
            addon_name="aws-ebs-csi-driver",
            resolve_conflicts_on_update="OVERWRITE",
            # Pod Identity association inline (sem criar objeto K8s separado)
            pod_identity_associations=[{
                "serviceAccount": "ebs-csi-controller-sa",
                "roleArn": ebs_csi_role_pod_id.role_arn,
            }],
        )

        # ──────────────────────────────────────────────────────────────
        # StorageClass gp3 como padrão (K8s manifest)
        # ──────────────────────────────────────────────────────────────
        cluster.add_manifest("StorageClassGp3", {
            "apiVersion": "storage.k8s.io/v1",
            "kind": "StorageClass",
            "metadata": {
                "name": "gp3",
                "annotations": {
                    "storageclass.kubernetes.io/is-default-class": "true"
                },
            },
            "provisioner": "ebs.csi.aws.com",
            "volumeBindingMode": "WaitForFirstConsumer",
            "reclaimPolicy": "Delete",
            "allowVolumeExpansion": True,
            "parameters": {
                "type": "gp3",
                "encrypted": "true",
                "kmsKeyId": ebs_kms_key.key_arn,
                # gp3 throughput e IOPS customizáveis
                "throughput": "125",   # MB/s (padrão gp3)
                "iops": "3000",        # IOPS (padrão gp3)
            },
        })

        # Remover gp2 como default (evitar conflito de dois defaults)
        cluster.add_manifest("PatchDefaultStorageClass", {
            "apiVersion": "storage.k8s.io/v1",
            "kind": "StorageClass",
            "metadata": {
                "name": "gp2",
                "annotations": {
                    "storageclass.kubernetes.io/is-default-class": "false"
                },
            },
            "provisioner": "kubernetes.io/aws-ebs",
            "volumeBindingMode": "WaitForFirstConsumer",
            "reclaimPolicy": "Delete",
        })

        # ──────────────────────────────────────────────────────────────
        # Opção B: IRSA (para Fargate pods ou ambientes não-EKS)
        # ──────────────────────────────────────────────────────────────
        # ebs_csi_sa = cluster.add_service_account("EbsCsiSA",
        #     name="ebs-csi-controller-sa",
        #     namespace="kube-system",
        # )
        # ebs_csi_sa.role.add_managed_policy(
        #     iam.ManagedPolicy.from_aws_managed_policy_name(
        #         "service-role/AmazonEBSCSIDriverPolicyV2"
        #     )
        # )
        # ebs_csi_addon = eks.CfnAddon(self, "EbsCsiAddon",
        #     cluster_name=cluster.cluster_name,
        #     addon_name="aws-ebs-csi-driver",
        #     service_account_role_arn=ebs_csi_sa.role.role_arn,
        #     resolve_conflicts_on_update="OVERWRITE",
        # )
```

---

## 6. Python — Operações de provisioning de storage

```python
"""
Aplicação que cria PVCs e demonstra o fluxo de dynamic provisioning.
Inclui cálculo de densidade de pods com Prefix Delegation.
"""
import math


# ──────────────────────────────────────────────────────────────────────
# Calculadora de densidade de pods por modo de CNI
# ──────────────────────────────────────────────────────────────────────

# Fonte: https://github.com/awslabs/amazon-eks-ami/blob/main/nodeadm/internal/kubelet/config.go
EC2_ENI_LIMITS = {
    "t3.small":   {"enis": 3,  "ips_per_eni": 4},
    "t3.medium":  {"enis": 3,  "ips_per_eni": 6},
    "m5.large":   {"enis": 3,  "ips_per_eni": 10},
    "m5.xlarge":  {"enis": 4,  "ips_per_eni": 15},
    "m5.2xlarge": {"enis": 4,  "ips_per_eni": 15},
    "m5.4xlarge": {"enis": 8,  "ips_per_eni": 30},
    "c5.large":   {"enis": 3,  "ips_per_eni": 10},
    "c5.xlarge":  {"enis": 4,  "ips_per_eni": 15},
    "c5.4xlarge": {"enis": 8,  "ips_per_eni": 30},
}
PREFIX_DELEGATION_BLOCK_SIZE = 16  # /28 CIDR


def calc_max_pods_secondary_ip(instance_type: str) -> int:
    """Calcula max pods em Secondary IP mode."""
    limits = EC2_ENI_LIMITS[instance_type]
    return (limits["enis"] * (limits["ips_per_eni"] - 1)) + 2


def calc_max_pods_prefix_delegation(
    instance_type: str,
    kubelet_max_pods: int = 110,
) -> dict:
    """
    Calcula max pods em Prefix Delegation mode.
    Retorna tanto o limite teórico da ENI quanto o limite do kubelet.
    """
    limits = EC2_ENI_LIMITS[instance_type]
    # Cada slot de IP vira um prefixo /28 de 16 IPs
    eni_capacity = (limits["enis"] * ((limits["ips_per_eni"] - 1) * PREFIX_DELEGATION_BLOCK_SIZE)) + 2
    effective_max = min(eni_capacity, kubelet_max_pods)
    return {
        "eni_theoretical_max": eni_capacity,
        "kubelet_cap":         kubelet_max_pods,
        "effective_max_pods":  effective_max,
        "density_multiplier":  round(effective_max / calc_max_pods_secondary_ip(instance_type), 1),
    }


def print_density_comparison():
    """Imprime tabela comparativa de densidade por instância."""
    header = f"{'Tipo':<14} {'SecIP':>6} {'PrefDel ENI':>12} {'PfDel(110)':>10} {'Ganho':>7}"
    print(header)
    print("-" * len(header))
    for itype in EC2_ENI_LIMITS:
        sec_ip = calc_max_pods_secondary_ip(itype)
        pref = calc_max_pods_prefix_delegation(itype, kubelet_max_pods=110)
        print(
            f"{itype:<14} {sec_ip:>6} {pref['eni_theoretical_max']:>12} "
            f"{pref['effective_max_pods']:>10} {pref['density_multiplier']:>6}x"
        )


# ──────────────────────────────────────────────────────────────────────
# Manifests de StatefulSet com PVC EBS (gerar via Python para injetar
# configuração dinâmica — ex: tamanho do volume por ambiente)
# ──────────────────────────────────────────────────────────────────────

def render_statefulset_with_ebs(
    name: str,
    namespace: str,
    replicas: int,
    storage_class: str = "gp3",
    volume_size_gi: int = 20,
    image: str = "postgres:16",
) -> dict:
    """
    Gera manifests de StatefulSet + PVC para workload stateful com EBS.
    O PVC é gerenciado pelo StatefulSet via volumeClaimTemplates.
    """
    return {
        "apiVersion": "apps/v1",
        "kind": "StatefulSet",
        "metadata": {"name": name, "namespace": namespace},
        "spec": {
            "serviceName": name,
            "replicas": replicas,
            "selector": {"matchLabels": {"app": name}},
            "podManagementPolicy": "OrderedReady",   # padrão: um pod de cada vez
            "template": {
                "metadata": {"labels": {"app": name}},
                "spec": {
                    "terminationGracePeriodSeconds": 30,
                    "containers": [{
                        "name": name,
                        "image": image,
                        "ports": [{"containerPort": 5432}],
                        "env": [
                            {"name": "POSTGRES_DB",       "value": "app"},
                            {"name": "POSTGRES_USER",     "value": "app"},
                            {"name": "POSTGRES_PASSWORD",
                             "valueFrom": {"secretKeyRef": {"name": f"{name}-secret", "key": "password"}}},
                        ],
                        "volumeMounts": [{
                            "name": "data",
                            "mountPath": "/var/lib/postgresql/data",
                            "subPath": "pgdata",   # evita perda de dados em restart
                        }],
                        "resources": {
                            "requests": {"cpu": "500m", "memory": "1Gi"},
                            "limits":   {"cpu": "2",    "memory": "4Gi"},
                        },
                        # Readiness probe — só considera pronto quando Postgres aceita conexões
                        "readinessProbe": {
                            "exec": {"command": ["pg_isready", "-U", "app"]},
                            "initialDelaySeconds": 10,
                            "periodSeconds": 5,
                        },
                    }],
                },
            },
            # volumeClaimTemplates: cria 1 PVC por réplica automaticamente
            # Nomes: data-<name>-0, data-<name>-1, ...
            "volumeClaimTemplates": [{
                "metadata": {"name": "data"},
                "spec": {
                    "accessModes": ["ReadWriteOnce"],   # EBS é sempre RWO
                    "storageClassName": storage_class,
                    "resources": {
                        "requests": {"storage": f"{volume_size_gi}Gi"}
                    },
                },
            }],
        },
    }


def render_pvc_snapshot(pvc_name: str, namespace: str, snapshot_class: str = "ebs-vsc") -> dict:
    """Gera VolumeSnapshot de um PVC EBS existente."""
    return {
        "apiVersion": "snapshot.storage.k8s.io/v1",
        "kind": "VolumeSnapshot",
        "metadata": {
            "name": f"{pvc_name}-snapshot",
            "namespace": namespace,
        },
        "spec": {
            "volumeSnapshotClassName": snapshot_class,
            "source": {
                "persistentVolumeClaimName": pvc_name
            },
        },
    }


if __name__ == "__main__":
    print("\n=== Densidade de Pods por Tipo de Instância ===\n")
    print_density_comparison()

    print("\n=== Detalhes m5.xlarge com Prefix Delegation ===")
    details = calc_max_pods_prefix_delegation("m5.xlarge", kubelet_max_pods=737)
    for k, v in details.items():
        print(f"  {k}: {v}")
```

---

## 7. CLI — Ciclo de vida completo de add-ons

```bash
# ═══════════════════════════════════════════════════════════════
# Introspection — descobrir versões e schema de configuração
# ═══════════════════════════════════════════════════════════════

CLUSTER="checkout-prod"
REGION="us-east-1"

# Listar versões disponíveis de um add-on para a versão K8s do cluster
K8S_VERSION=$(aws eks describe-cluster --name "$CLUSTER" \
  --query 'cluster.version' --output text)

aws eks describe-addon-versions \
  --addon-name amazon-vpc-cni \
  --kubernetes-version "$K8S_VERSION" \
  --query 'addons[0].addonVersions[*].{Version:addonVersion,Default:compatibilities[0].defaultVersion}' \
  --output table

# Ver schema de configuração disponível (campos configuráveis via --configuration-values)
aws eks describe-addon-configuration \
  --addon-name amazon-vpc-cni \
  --addon-version v1.21.1-eksbuild.8

# Ver namespace padrão do add-on
aws eks describe-addon-versions \
  --addon-name aws-ebs-csi-driver \
  --query "addons[].defaultNamespace"

# Listar add-ons instalados no cluster
aws eks list-addons --cluster-name "$CLUSTER" --output table

# Detalhar add-on instalado (status, versão, health)
aws eks describe-addon \
  --cluster-name "$CLUSTER" \
  --addon-name amazon-vpc-cni \
  --query 'addon.{Status:status,Version:addonVersion,Issues:health.issues}'

# ═══════════════════════════════════════════════════════════════
# VPC CNI — habilitar Prefix Delegation
# ═══════════════════════════════════════════════════════════════

# Opção A: via update-addon com --configuration-values
aws eks update-addon \
  --cluster-name "$CLUSTER" \
  --addon-name amazon-vpc-cni \
  --resolve-conflicts PRESERVE \
  --configuration-values '{"env":{"ENABLE_PREFIX_DELEGATION":"true","WARM_PREFIX_TARGET":"1"}}'

# Verificar que a env var foi aplicada no DaemonSet
kubectl get daemonset aws-node -n kube-system \
  -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="ENABLE_PREFIX_DELEGATION")].value}'
# Esperado: true

# Verificar pods com prefixos atribuídos
kubectl describe node <node-name> | grep -A 20 "Addresses:"
# Procurar por entradas com /28 no campo de IPs

# Opção B: via kubectl (campo não gerenciado pelo EKS)
kubectl set env daemonset aws-node -n kube-system \
  ENABLE_PREFIX_DELEGATION=true \
  WARM_PREFIX_TARGET=1
# ATENÇÃO: este campo não é gerenciado pelo EKS (não será sobrescrito),
# mas perde-se rastreabilidade — preferir --configuration-values

# ═══════════════════════════════════════════════════════════════
# CoreDNS — escalar e customizar
# ═══════════════════════════════════════════════════════════════

# Verificar estado atual
kubectl get deployment coredns -n kube-system
kubectl top pod -n kube-system -l k8s-app=kube-dns

# Atualizar CoreDNS com 3 réplicas via add-on
aws eks update-addon \
  --cluster-name "$CLUSTER" \
  --addon-name coredns \
  --resolve-conflicts PRESERVE \
  --configuration-values '{"replicaCount":3}'

# Aguardar atualização
aws eks wait addon-active \
  --cluster-name "$CLUSTER" \
  --addon-name coredns

# Verificar
kubectl get pods -n kube-system -l k8s-app=kube-dns

# ═══════════════════════════════════════════════════════════════
# EBS CSI Driver — instalação e StorageClass
# ═══════════════════════════════════════════════════════════════

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
EBS_ROLE="AmazonEKS_EBS_CSI_DriverRole"

# Step 1: IAM role para o driver (via Pod Identity)
cat > ebs-pod-identity-trust.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "pods.eks.amazonaws.com" },
    "Action": ["sts:AssumeRole", "sts:TagSession"]
  }]
}
EOF

aws iam create-role \
  --role-name "$EBS_ROLE" \
  --assume-role-policy-document file://ebs-pod-identity-trust.json

aws iam attach-role-policy \
  --role-name "$EBS_ROLE" \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicyV2

# Step 2: Instalar o add-on com Pod Identity inline
aws eks create-addon \
  --cluster-name "$CLUSTER" \
  --addon-name aws-ebs-csi-driver \
  --resolve-conflicts OVERWRITE \
  --pod-identity-associations \
    "serviceAccount=ebs-csi-controller-sa,roleArn=arn:aws:iam::${ACCOUNT_ID}:role/${EBS_ROLE}"

# Aguardar ACTIVE
aws eks wait addon-active \
  --cluster-name "$CLUSTER" \
  --addon-name aws-ebs-csi-driver

# Verificar pods do driver
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver
# Esperado: ebs-csi-controller-* (Deployment, 2 réplicas) + ebs-csi-node-* (DaemonSet)

# Step 3: Criar StorageClass gp3
kubectl apply -f - << 'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
allowVolumeExpansion: true
parameters:
  type: gp3
  encrypted: "true"
EOF

# Remover gp2 como default
kubectl patch storageclass gp2 \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'

# Verificar StorageClasses
kubectl get storageclass
# Esperado: gp3 (default) e gp2 (sem default)

# Step 4: Testar dynamic provisioning com PVC de teste
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-test-pvc
  namespace: default
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: gp3
  resources:
    requests:
      storage: 5Gi
EOF

# O PVC fica Pending até um pod referenciar ele (WaitForFirstConsumer)
kubectl get pvc ebs-test-pvc

# Criar pod de teste para trigger o provisionamento
kubectl run ebs-test --image=busybox \
  --restart=Never \
  --overrides='{"spec":{"volumes":[{"name":"data","persistentVolumeClaim":{"claimName":"ebs-test-pvc"}}],"containers":[{"name":"ebs-test","image":"busybox","command":["sh","-c","echo ok > /data/test && cat /data/test && sleep 60"],"volumeMounts":[{"name":"data","mountPath":"/data"}]}]}}' \
  -- sh -c "echo ok"

# Verificar que o PV foi criado e o volume EBS provisionado
kubectl get pv
# Deve mostrar um PV com STORAGECLASS=gp3 e STATUS=Bound

kubectl describe pvc ebs-test-pvc | grep -E "Volume|StorageClass|Status"

# Ver volume EBS criado na AWS
kubectl get pv -o jsonpath='{.items[*].spec.csi.volumeHandle}' | tr ' ' '\n' | \
  while read volid; do
    aws ec2 describe-volumes --volume-ids "$volid" \
      --query 'Volumes[0].{ID:VolumeId,Type:VolumeType,Size:Size,AZ:AvailabilityZone}'
  done

# Limpar teste
kubectl delete pod ebs-test
kubectl delete pvc ebs-test-pvc   # deletar PVC apaga o PV e o volume EBS (reclaimPolicy: Delete)

# ═══════════════════════════════════════════════════════════════
# Verificar health de todos os add-ons
# ═══════════════════════════════════════════════════════════════

aws eks list-addons --cluster-name "$CLUSTER" --output json | \
  jq -r '.addons[]' | \
  while read addon; do
    status=$(aws eks describe-addon \
      --cluster-name "$CLUSTER" \
      --addon-name "$addon" \
      --query 'addon.{Status:status,Issues:health.issues}' \
      --output json)
    echo "=== $addon ==="
    echo "$status"
  done
```

---

## 8. Armadilhas (Pitfalls)

[FATO] **`resolve-conflicts OVERWRITE` em update sobrescreve customizações**: se você editou diretamente um campo gerenciado pelo EKS (ex: `env` do aws-node via `kubectl set env`) e depois executa `update-addon` com `OVERWRITE`, suas customizações são perdidas. Use `PRESERVE` em updates de clusters com configuração customizada.

[FATO] **Dois StorageClasses marcados como `default` causam comportamento imprevisível**: quando um PVC não especifica `storageClassName`, o K8s usa o default. Se há dois defaults, o admission controller do K8s retorna erro. Sempre garantir que apenas um SC tenha a annotation `is-default-class: "true"`.

[FATO] **`volumeBindingMode: Immediate` com StatefulSets multi-AZ**: o PVC é provisionado imediatamente na AZ onde o K8s schedula (aleatoriamente). Se o pod for reiniciado em outra AZ, o mount falha porque EBS é por-AZ. `WaitForFirstConsumer` resolve isso ao criar o volume na AZ do nó onde o pod foi agendado.

[FATO] **EBS CSI node DaemonSet não roda em Fargate**: se você tem um cluster com nós EC2 + Fargate, o node DaemonSet (`ebs-csi-node`) é agendado apenas em EC2. Pods Fargate não podem usar PVCs EBS independentemente. Use EFS CSI Driver para storage compartilhado com suporte a Fargate.

[FATO] **Prefix Delegation: downgrade de VPC CNI é bloqueado**: uma vez que `ENABLE_PREFIX_DELEGATION=true` está ativo e nós receberam prefixos /28, não é possível fazer downgrade do VPC CNI para versão < 1.9.0 sem remover todos os nós do cluster.

[FATO] **EBS volumes e Multi-Attach**: volumes `gp3` e `gp2` suportam apenas `ReadWriteOnce` (1 nó por vez). `ReadWriteMany` não é suportado com EBS; para isso, usar EFS ou FSx for Lustre.

[CONSENSO] **Prefix Delegation: criar novos node groups em vez de rolling replace**: nós existentes com IPs individuais atribuídos e novos prefixos atribuídos no mesmo nó (durante transição) podem causar inconsistência na capacidade reportada pelo kubelet. O caminho seguro é provisionar novos node groups com Prefix Delegation habilitado e drenar os antigos.

---

## Exercício de Reflexão

Um cluster EKS (`m5.xlarge`, 10 nós) está atingindo o limite de pods. A subnet VPC tem /22 (1022 IPs usáveis). O kubelet está configurado com o default de 110 pods/nó.

1. Em **Secondary IP Mode**, qual o máximo teórico de pods por nó em um m5.xlarge? (use a fórmula). Com 10 nós, quantos pods total? Quantos IPs de VPC são consumidos (incluindo 1 IP por ENI para o nó e 1 por pod)?

2. Em **Prefix Delegation Mode** com `WARM_PREFIX_TARGET=1` e kubelet em 110: qual o max pods/nó? O gargalo é a ENI ou o kubelet? Como aumentar o kubelet cap e qual script AWS fornece para calcular o valor correto?

3. O time quer migrar de Secondary IP para Prefix Delegation sem downtime. Descreva a sequência correta de passos. Por que a abordagem de rolling replace dos nós existentes é arriscada?

4. Você quer que pods do namespace `databases` usem volumes EBS criptografados com uma CMK específica (não a `aws/ebs`). Quais permissões adicionais o role do EBS CSI Driver precisa? Onde essas permissões devem ser configuradas (policy on the key? policy on the role?)?

5. Um desenvolvedor cria um PVC com `accessModes: ReadWriteMany` usando a StorageClass `gp3`. O que acontece e por quê? Qual seria o CSI driver correto para storage `ReadWriteMany` em um workload EKS?

---

## Referências

- [FATO] [Amazon EKS add-ons](https://docs.aws.amazon.com/eks/latest/userguide/eks-add-ons.html) — docs.aws.amazon.com
- [FATO] [Assign more IP addresses to Amazon EKS nodes with prefixes](https://docs.aws.amazon.com/eks/latest/userguide/cni-increase-ip-addresses.html) — docs.aws.amazon.com
- [FATO] [Use Kubernetes volume storage with Amazon EBS](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html) — docs.aws.amazon.com
- [FATO] [create-addon — AWS CLI Reference](https://docs.aws.amazon.com/cli/latest/reference/eks/create-addon.html) — docs.aws.amazon.com
- [FATO] [Prefix Delegation mode for Linux — EKS Best Practices](https://docs.aws.amazon.com/eks/latest/best-practices/prefix-mode-linux.html) — docs.aws.amazon.com
- [FATO] [aws-ebs-csi-driver GitHub — AmazonEBSCSIDriverPolicyV2 migration](https://github.com/kubernetes-sigs/aws-ebs-csi-driver/issues/2918) — github.com
