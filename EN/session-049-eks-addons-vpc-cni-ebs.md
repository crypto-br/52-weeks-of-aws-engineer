# Session 049 — EKS: Managed Add-ons — VPC CNI, CoreDNS and EBS CSI Driver

**Estimated duration:** 60 min  
**Prerequisite:** session-048 (IRSA and Pod Identity)

---

## Session Objectives

- Understand the EKS managed add-ons model: categories, conflicts, field management
- Install, configure and update add-ons via CLI using `--configuration-values` and `--resolve-conflicts`
- Configure VPC CNI for Prefix Delegation and calculate the pod density gain
- Install the EBS CSI Driver with correct IAM (IRSA or Pod Identity) and provision dynamic PersistentVolumes

---

## 1. Managed Add-ons Model

### 1.1 Add-on Categories

[FACT] There are three categories of add-ons in EKS, with distinct support levels:

```
╔══════════════════════╦════════════════════════════════╦══════════════╗
║ Category             ║ Examples                       ║ Support      ║
╠══════════════════════╬════════════════════════════════╬══════════════╣
║ AWS Add-ons          ║ VPC CNI, CoreDNS, kube-proxy,  ║ Full AWS     ║
║                      ║ EBS CSI, Pod Identity Agent,   ║ Support      ║
║                      ║ EFS CSI, S3 CSI                ║              ║
╠══════════════════════╬════════════════════════════════╬══════════════╣
║ AWS Marketplace      ║ Splunk, Datadog, Dynatrace,    ║ Partner      ║
║ Add-ons              ║ Tetrate, Calico                ║ Support      ║
╠══════════════════════╬════════════════════════════════╬══════════════╣
║ Community Add-ons    ║ Metrics Server, cert-manager,  ║ Community    ║
║                      ║ external-dns, Argo CD          ║ (AWS tests   ║
║                      ║                                ║  K8s compat) ║
╚══════════════════════╩════════════════════════════════╩══════════════╝
```

[FACT] The three add-ons **auto-installed** by EKS on every new cluster:
- `amazon-vpc-cni` (VPC CNI) — manages ENIs and pod IPs
- `coredns` — cluster internal DNS
- `kube-proxy` — Service network routing

When created via Console, these three are installed as **managed** add-ons. When created via `eksctl` without a `config` file, they are installed as **self-managed** (without EKS management).

### 1.2 Kubernetes Field Management (Server-Side Apply)

[FACT] EKS uses **Kubernetes Server-Side Apply** (SSA) to manage add-on fields. This means:

```
┌──────────────────────────────────────────────────────────────┐
│ Field managed by EKS (field manager: eks)                    │
│ → EKS overwrites if you change via kubectl apply             │
│                                                              │
│ Field NOT managed by EKS                                     │
│ → You can change via kubectl; EKS does not overwrite         │
└──────────────────────────────────────────────────────────────┘
```

To see which fields an add-on can configure via `--configuration-values`:

```bash
aws eks describe-addon-configuration \
  --addon-name amazon-vpc-cni \
  --addon-version v1.21.1-eksbuild.8 \
  --query 'schema' \
  --output text | python3 -m json.tool
```

[FACT] The `eks:addon-manager` has a `ClusterRoleBinding` called `eks:addon-cluster-admin` that binds the `cluster-admin` role to the `eks:addon-manager` identity. If this `ClusterRoleBinding` is removed, EKS loses the ability to manage add-ons.

### 1.3 Resolve Conflicts

[FACT] The `--resolve-conflicts` parameter controls what happens when the EKS add-on conflicts with existing configurations:

```
OVERWRITE  → EKS overwrites with default values (use on new clusters)
PRESERVE   → EKS keeps existing values (use on updates with custom config)
NONE       → EKS changes nothing; creation may fail if there is a conflict
```

[FACT] When **updating** an add-on with `--resolve-conflicts PRESERVE`, EKS keeps the fields you customized and only updates fields that were never modified.

---
## 2. VPC CNI — Prefix Delegation

### 2.1 Recap: Secondary IP Mode vs Prefix Delegation

[FACT] In Secondary IP Mode (default), each secondary IP slot on an ENI receives **1 IP address**, which is assigned to a pod. The max pods formula is:

```
max_pods = (num_ENIs × (IPs_per_ENI - 1)) + 2

Example m5.xlarge: 4 ENIs × (15 - 1) + 2 = 58 pods
```

[FACT] In **Prefix Delegation Mode**, each slot receives a **/28 prefix (16 IPs)** instead of 1 IP. This multiplies density by up to 16×:

```
max_pods = (num_ENIs × ((IPs_per_ENI - 1) × 16)) + 2

Example m5.xlarge: 4 × (14 × 16) + 2 = 898 pods (limited by kubelet)
```

[FACT] Prefix Delegation is available only on **Nitro** instances and requires VPC CNI ≥ v1.9.0.

[FACT] Migration from Secondary IP to Prefix Delegation is **irreversible per node**. The AWS recommendation is to create new node groups instead of doing a rolling replace of existing ones. A node with a mix of IPs and prefixes may report inconsistent capacity.

### 2.2 Warm Pool Control Variables

[FACT] With Prefix Delegation enabled, the relevant control variables are:

```
ENABLE_PREFIX_DELEGATION=true   → Enables prefix delegation (required)
WARM_PREFIX_TARGET=1            → Keeps 1 "warm" /28 prefix per ENI (default)
WARM_IP_TARGET=N                → (alternative) keeps N individual warm IPs
MINIMUM_IP_TARGET=N             → Ensures minimum of N IPs available in the pool
```

[FACT] `WARM_PREFIX_TARGET` and `WARM_IP_TARGET` are **mutually exclusive** as the primary strategy. Use `WARM_PREFIX_TARGET` for fast burst; use `WARM_IP_TARGET` for fine-grained cost control (fewer idle IPs).

### 2.3 Per-Node Pod Limit — Required Adjustment

[FACT] The kubelet default is 110 pods per node. With Prefix Delegation, you need to explicitly increase this limit via `maxPodsPerNode` in the launch template or the kubelet `--max-pods` field. The AWS `max-pods-calculator.sh` script calculates the correct value per instance type.

---
## 3. CoreDNS — Configuration and Scaling

[FACT] CoreDNS runs by default with **2 replicas** in the `kube-system` namespace. The managed add-on exposes resource configuration via `--configuration-values`:

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

[FACT] For production environments with high DNS resolution volume, the default of 2 replicas may be insufficient. Scaling to 3–5 replicas is common practice in clusters with hundreds of pods. CoreDNS can also be scaled with HPA based on memory or CPU.

[FACT] If you customize the `Corefile` directly via `kubectl edit configmap -n kube-system coredns`, that field is **not** managed by EKS (SSA), so EKS does not overwrite it. To include it in version control, use `--configuration-values` with the `corefile` key.

---

## 4. EBS CSI Driver

### 4.1 What the EBS CSI Driver Does

[FACT] The `aws-ebs-csi-driver` is the CSI (Container Storage Interface) plugin that manages the lifecycle of Amazon EBS volumes as Kubernetes `PersistentVolumes`. It enables:
- Dynamically provisioning EBS volumes when creating PVCs
- Online volume resizing (`allowVolumeExpansion: true`)
- Taking EBS snapshots via `VolumeSnapshot`
- Encrypting volumes with KMS customer-managed keys

[FACT] Driver components:
- **Controller** (Deployment, 2 replicas): provisions, attaches, detaches volumes; can run on Fargate
- **Node** (DaemonSet): mounts/unmounts volumes on the node; runs only on **EC2** (not Fargate)

[FACT] EBS is not supported on Fargate pods (only EFS is supported on Fargate). Fargate pods cannot mount EBS PVCs.

### 4.2 IAM — Three Available Policies

[FACT] The EBS CSI Driver requires IAM to call EC2 APIs (CreateVolume, AttachVolume, etc.). There are three managed policy options:

```
AmazonEBSCSIDriverPolicyV2          → Tag-based restriction (recommended)
                                       Allows creating volumes only with
                                       tag kubernetes.io/cluster/<name>

AmazonEBSCSIDriverEKSClusterScopedPolicy → Cluster ARN restriction
                                           More restrictive for multi-cluster

AmazonEBSCSIDriverPolicy            → No scope restriction (legacy)
                                       Avoid in new deployments
```

[FACT] The `AmazonEBSCSIDriverPolicyV2` policy requires the `StorageClass` to include the `kubernetes.io/cluster/<cluster-name>` tag on created volumes (the driver adds this automatically when the add-on is properly installed).

### 4.3 Default StorageClass

[FACT] The `aws-ebs-csi-driver` creates a default `StorageClass` called `gp2` (legacy) or `gp3` (recommended). To use `gp3` as default:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer    # CRITICAL: avoids cross-AZ binding
reclaimPolicy: Delete
allowVolumeExpansion: true
parameters:
  type: gp3
  encrypted: "true"
  # kmsKeyId: arn:aws:kms:...  # optional: CMK
```

[FACT] `volumeBindingMode: WaitForFirstConsumer` is critical: it defers EBS volume provisioning until a pod is scheduled on a specific node. This ensures the volume is created in the **same AZ as the node**. With `Immediate`, the volume may be created in a different AZ than the pod, causing a mount failure.

### 4.4 Snapshot Controller

[FACT] `VolumeSnapshot` support requires separate installation of the **CSI Snapshot Controller** — it is **not** included in the `aws-ebs-csi-driver`. The snapshot controller is a CRD + controller:
- `VolumeSnapshotClass`
- `VolumeSnapshot`
- `VolumeSnapshotContent`

The `aws-ebs-csi-driver` add-on includes a `csi-snapshotter` sidecar that communicates with the snapshot controller, but the controller itself must be installed via a community add-on or Helm.

---
## 5. CDK Python — Add-ons and EBS CSI Driver

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
    Manages EKS add-ons: VPC CNI (Prefix Delegation), CoreDNS and EBS CSI Driver.
    """
    def __init__(self, scope: Construct, construct_id: str,
                 cluster: eks.Cluster, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # ──────────────────────────────────────────────────────────────
        # VPC CNI — enable Prefix Delegation via configuration-values
        # Requires that the add-on already exists; uses CfnAddon (L1) to
        # access configuration_values
        # ──────────────────────────────────────────────────────────────
        vpc_cni_config = {
            "env": {
                "ENABLE_PREFIX_DELEGATION": "true",
                "WARM_PREFIX_TARGET":       "1",
                # For fine-grained warm pool control (alternative to WARM_PREFIX_TARGET):
                # "WARM_IP_TARGET":    "16",
                # "MINIMUM_IP_TARGET": "32",
                "AWS_VPC_K8S_CNI_LOGLEVEL": "WARN",   # reduce log noise
            },
            # Resource limits for the aws-node DaemonSet
            "resources": {
                "requests": {"cpu": "25m"},
                "limits":   {"cpu": "100m"},
            },
        }

        import json
        vpc_cni_addon = eks.CfnAddon(self, "VpcCniAddon",
            cluster_name=cluster.cluster_name,
            addon_name="amazon-vpc-cni",
            # Do not pin version → EKS uses the default version for the cluster K8s version
            # addon_version="v1.21.1-eksbuild.8",
            resolve_conflicts_on_update="PRESERVE",   # keep customizations
            configuration_values=json.dumps(vpc_cni_config),
        )

        # ──────────────────────────────────────────────────────────────
        # CoreDNS — 3 replicas in production, adjusted resources
        # ──────────────────────────────────────────────────────────────
        coredns_config = {
            "replicaCount": 3,
            "resources": {
                "limits":   {"cpu": "200m", "memory": "256Mi"},
                "requests": {"cpu": "100m", "memory": "70Mi"},
            },
            # podDisruptionBudget ensures minimum of 1 replica during drain
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
        # EBS CSI Driver — IAM via Pod Identity (preferred) or IRSA
        # ──────────────────────────────────────────────────────────────

        # Option A: Pod Identity (recommended for EKS EC2 nodes)
        ebs_csi_role_pod_id = iam.Role(self, "EbsCsiRolePodId",
            role_name="AmazonEKS_EBS_CSI_DriverRole",
            description="EBS CSI Driver role via Pod Identity",
            assumed_by=iam.ServicePrincipal("pods.eks.amazonaws.com"),
        )
        # TagSession required for Pod Identity
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

        # KMS key for volume encryption (optional)
        ebs_kms_key = kms.Key(self, "EbsKmsKey",
            description="CMK for EBS volumes in the EKS cluster",
            enable_key_rotation=True,
        )
        # Additional KMS permissions for the driver role
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

        # EBS CSI add-on with inline Pod Identity (pod-identity-associations)
        ebs_csi_addon = eks.CfnAddon(self, "EbsCsiAddon",
            cluster_name=cluster.cluster_name,
            addon_name="aws-ebs-csi-driver",
            resolve_conflicts_on_update="OVERWRITE",
            # Inline Pod Identity association (no separate K8s object needed)
            pod_identity_associations=[{
                "serviceAccount": "ebs-csi-controller-sa",
                "roleArn": ebs_csi_role_pod_id.role_arn,
            }],
        )

        # ──────────────────────────────────────────────────────────────
        # StorageClass gp3 as default (K8s manifest)
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
                # gp3 customizable throughput and IOPS
                "throughput": "125",   # MB/s (gp3 default)
                "iops": "3000",        # IOPS (gp3 default)
            },
        })

        # Remove gp2 as default (avoid conflict of two defaults)
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
        # Option B: IRSA (for Fargate pods or non-EKS environments)
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
## 6. Python — Storage Provisioning Operations

```python
"""
Application that creates PVCs and demonstrates the dynamic provisioning flow.
Includes pod density calculation with Prefix Delegation.
"""
import math


# ──────────────────────────────────────────────────────────────────────
# Pod density calculator by CNI mode
# ──────────────────────────────────────────────────────────────────────

# Source: https://github.com/awslabs/amazon-eks-ami/blob/main/nodeadm/internal/kubelet/config.go
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
    """Calculates max pods in Secondary IP mode."""
    limits = EC2_ENI_LIMITS[instance_type]
    return (limits["enis"] * (limits["ips_per_eni"] - 1)) + 2


def calc_max_pods_prefix_delegation(
    instance_type: str,
    kubelet_max_pods: int = 110,
) -> dict:
    """
    Calculates max pods in Prefix Delegation mode.
    Returns both the theoretical ENI limit and the kubelet limit.
    """
    limits = EC2_ENI_LIMITS[instance_type]
    # Each IP slot becomes a /28 prefix of 16 IPs
    eni_capacity = (limits["enis"] * ((limits["ips_per_eni"] - 1) * PREFIX_DELEGATION_BLOCK_SIZE)) + 2
    effective_max = min(eni_capacity, kubelet_max_pods)
    return {
        "eni_theoretical_max": eni_capacity,
        "kubelet_cap":         kubelet_max_pods,
        "effective_max_pods":  effective_max,
        "density_multiplier":  round(effective_max / calc_max_pods_secondary_ip(instance_type), 1),
    }


def print_density_comparison():
    """Prints a comparative density table by instance type."""
    header = f"{'Type':<14} {'SecIP':>6} {'PrefDel ENI':>12} {'PfDel(110)':>10} {'Gain':>7}"
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
# StatefulSet manifests with EBS PVC (generated via Python to inject
# dynamic configuration — e.g.: volume size per environment)
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
    Generates StatefulSet + PVC manifests for a stateful workload with EBS.
    The PVC is managed by the StatefulSet via volumeClaimTemplates.
    """
    return {
        "apiVersion": "apps/v1",
        "kind": "StatefulSet",
        "metadata": {"name": name, "namespace": namespace},
        "spec": {
            "serviceName": name,
            "replicas": replicas,
            "selector": {"matchLabels": {"app": name}},
            "podManagementPolicy": "OrderedReady",   # default: one pod at a time
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
                            "subPath": "pgdata",   # prevents data loss on restart
                        }],
                        "resources": {
                            "requests": {"cpu": "500m", "memory": "1Gi"},
                            "limits":   {"cpu": "2",    "memory": "4Gi"},
                        },
                        # Readiness probe — only considers ready when Postgres accepts connections
                        "readinessProbe": {
                            "exec": {"command": ["pg_isready", "-U", "app"]},
                            "initialDelaySeconds": 10,
                            "periodSeconds": 5,
                        },
                    }],
                },
            },
            # volumeClaimTemplates: creates 1 PVC per replica automatically
            # Names: data-<name>-0, data-<name>-1, ...
            "volumeClaimTemplates": [{
                "metadata": {"name": "data"},
                "spec": {
                    "accessModes": ["ReadWriteOnce"],   # EBS is always RWO
                    "storageClassName": storage_class,
                    "resources": {
                        "requests": {"storage": f"{volume_size_gi}Gi"}
                    },
                },
            }],
        },
    }


def render_pvc_snapshot(pvc_name: str, namespace: str, snapshot_class: str = "ebs-vsc") -> dict:
    """Generates a VolumeSnapshot of an existing EBS PVC."""
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
    print("\n=== Pod Density by Instance Type ===\n")
    print_density_comparison()

    print("\n=== m5.xlarge Details with Prefix Delegation ===")
    details = calc_max_pods_prefix_delegation("m5.xlarge", kubelet_max_pods=737)
    for k, v in details.items():
        print(f"  {k}: {v}")
```

---
## 7. CLI — Complete Add-on Lifecycle

```bash
# ═══════════════════════════════════════════════════════════════
# Introspection — discover versions and configuration schema
# ═══════════════════════════════════════════════════════════════

CLUSTER="checkout-prod"
REGION="us-east-1"

# List available versions of an add-on for the cluster K8s version
K8S_VERSION=$(aws eks describe-cluster --name "$CLUSTER" \
  --query 'cluster.version' --output text)

aws eks describe-addon-versions \
  --addon-name amazon-vpc-cni \
  --kubernetes-version "$K8S_VERSION" \
  --query 'addons[0].addonVersions[*].{Version:addonVersion,Default:compatibilities[0].defaultVersion}' \
  --output table

# View available configuration schema (configurable fields via --configuration-values)
aws eks describe-addon-configuration \
  --addon-name amazon-vpc-cni \
  --addon-version v1.21.1-eksbuild.8

# View default namespace of the add-on
aws eks describe-addon-versions \
  --addon-name aws-ebs-csi-driver \
  --query "addons[].defaultNamespace"

# List add-ons installed on the cluster
aws eks list-addons --cluster-name "$CLUSTER" --output table

# Detail installed add-on (status, version, health)
aws eks describe-addon \
  --cluster-name "$CLUSTER" \
  --addon-name amazon-vpc-cni \
  --query 'addon.{Status:status,Version:addonVersion,Issues:health.issues}'

# ═══════════════════════════════════════════════════════════════
# VPC CNI — enable Prefix Delegation
# ═══════════════════════════════════════════════════════════════

# Option A: via update-addon with --configuration-values
aws eks update-addon \
  --cluster-name "$CLUSTER" \
  --addon-name amazon-vpc-cni \
  --resolve-conflicts PRESERVE \
  --configuration-values '{"env":{"ENABLE_PREFIX_DELEGATION":"true","WARM_PREFIX_TARGET":"1"}}'

# Verify that the env var was applied to the DaemonSet
kubectl get daemonset aws-node -n kube-system \
  -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="ENABLE_PREFIX_DELEGATION")].value}'
# Expected: true

# Verify pods with assigned prefixes
kubectl describe node <node-name> | grep -A 20 "Addresses:"
# Look for entries with /28 in the IP field

# Option B: via kubectl (field not managed by EKS)
kubectl set env daemonset aws-node -n kube-system \
  ENABLE_PREFIX_DELEGATION=true \
  WARM_PREFIX_TARGET=1
# WARNING: this field is not managed by EKS (will not be overwritten),
# but traceability is lost — prefer --configuration-values

# ═══════════════════════════════════════════════════════════════
# CoreDNS — scale and customize
# ═══════════════════════════════════════════════════════════════

# Check current state
kubectl get deployment coredns -n kube-system
kubectl top pod -n kube-system -l k8s-app=kube-dns

# Update CoreDNS with 3 replicas via add-on
aws eks update-addon \
  --cluster-name "$CLUSTER" \
  --addon-name coredns \
  --resolve-conflicts PRESERVE \
  --configuration-values '{"replicaCount":3}'

# Wait for update
aws eks wait addon-active \
  --cluster-name "$CLUSTER" \
  --addon-name coredns

# Verify
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

```bash
# ═══════════════════════════════════════════════════════════════
# EBS CSI Driver — installation and StorageClass
# ═══════════════════════════════════════════════════════════════

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
EBS_ROLE="AmazonEKS_EBS_CSI_DriverRole"

# Step 1: IAM role for the driver (via Pod Identity)
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

# Step 2: Install the add-on with inline Pod Identity
aws eks create-addon \
  --cluster-name "$CLUSTER" \
  --addon-name aws-ebs-csi-driver \
  --resolve-conflicts OVERWRITE \
  --pod-identity-associations \
    "serviceAccount=ebs-csi-controller-sa,roleArn=arn:aws:iam::${ACCOUNT_ID}:role/${EBS_ROLE}"

# Wait for ACTIVE
aws eks wait addon-active \
  --cluster-name "$CLUSTER" \
  --addon-name aws-ebs-csi-driver

# Verify driver pods
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver
# Expected: ebs-csi-controller-* (Deployment, 2 replicas) + ebs-csi-node-* (DaemonSet)

# Step 3: Create StorageClass gp3
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

# Remove gp2 as default
kubectl patch storageclass gp2 \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'

# Verify StorageClasses
kubectl get storageclass
# Expected: gp3 (default) and gp2 (without default)

# Step 4: Test dynamic provisioning with a test PVC
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

# The PVC stays Pending until a pod references it (WaitForFirstConsumer)
kubectl get pvc ebs-test-pvc

# Create test pod to trigger provisioning
kubectl run ebs-test --image=busybox \
  --restart=Never \
  --overrides='{"spec":{"volumes":[{"name":"data","persistentVolumeClaim":{"claimName":"ebs-test-pvc"}}],"containers":[{"name":"ebs-test","image":"busybox","command":["sh","-c","echo ok > /data/test && cat /data/test && sleep 60"],"volumeMounts":[{"name":"data","mountPath":"/data"}]}]}}' \
  -- sh -c "echo ok"

# Verify that the PV was created and the EBS volume provisioned
kubectl get pv
# Should show a PV with STORAGECLASS=gp3 and STATUS=Bound

kubectl describe pvc ebs-test-pvc | grep -E "Volume|StorageClass|Status"

# View EBS volume created in AWS
kubectl get pv -o jsonpath='{.items[*].spec.csi.volumeHandle}' | tr ' ' '\n' | \
  while read volid; do
    aws ec2 describe-volumes --volume-ids "$volid" \
      --query 'Volumes[0].{ID:VolumeId,Type:VolumeType,Size:Size,AZ:AvailabilityZone}'
  done

# Clean up test
kubectl delete pod ebs-test
kubectl delete pvc ebs-test-pvc   # deleting PVC removes the PV and the EBS volume (reclaimPolicy: Delete)

# ═══════════════════════════════════════════════════════════════
# Verify health of all add-ons
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
## 8. Pitfalls

[FACT] **`resolve-conflicts OVERWRITE` on update overwrites customizations**: if you directly edited a field managed by EKS (e.g., `env` of aws-node via `kubectl set env`) and then run `update-addon` with `OVERWRITE`, your customizations are lost. Use `PRESERVE` on updates to clusters with custom configuration.

[FACT] **Two StorageClasses marked as `default` cause unpredictable behavior**: when a PVC does not specify `storageClassName`, K8s uses the default. If there are two defaults, the K8s admission controller returns an error. Always ensure that only one SC has the `is-default-class: "true"` annotation.

[FACT] **`volumeBindingMode: Immediate` with multi-AZ StatefulSets**: the PVC is provisioned immediately in the AZ where K8s schedules (randomly). If the pod is restarted in another AZ, the mount fails because EBS is per-AZ. `WaitForFirstConsumer` solves this by creating the volume in the AZ of the node where the pod was scheduled.

[FACT] **EBS CSI node DaemonSet does not run on Fargate**: if you have a cluster with EC2 + Fargate nodes, the node DaemonSet (`ebs-csi-node`) is scheduled only on EC2. Fargate pods cannot use EBS PVCs regardless. Use the EFS CSI Driver for shared storage with Fargate support.

[FACT] **Prefix Delegation: VPC CNI downgrade is blocked**: once `ENABLE_PREFIX_DELEGATION=true` is active and nodes have received /28 prefixes, it is not possible to downgrade VPC CNI to version < 1.9.0 without removing all nodes from the cluster.

[FACT] **EBS volumes and Multi-Attach**: `gp3` and `gp2` volumes support only `ReadWriteOnce` (1 node at a time). `ReadWriteMany` is not supported with EBS; for that, use EFS or FSx for Lustre.

[CONSENSUS] **Prefix Delegation: create new node groups instead of rolling replace**: existing nodes with individually assigned IPs and new prefixes assigned on the same node (during transition) can cause inconsistency in the capacity reported by the kubelet. The safe path is to provision new node groups with Prefix Delegation enabled and drain the old ones.

---

## Reflection Exercise

An EKS cluster (`m5.xlarge`, 10 nodes) is hitting the pod limit. The VPC subnet has /22 (1022 usable IPs). The kubelet is configured with the default of 110 pods/node.

1. In **Secondary IP Mode**, what is the theoretical maximum pods per node on an m5.xlarge? (use the formula). With 10 nodes, how many total pods? How many VPC IPs are consumed (including 1 IP per ENI for the node and 1 per pod)?

2. In **Prefix Delegation Mode** with `WARM_PREFIX_TARGET=1` and kubelet at 110: what is the max pods/node? Is the bottleneck the ENI or the kubelet? How do you increase the kubelet cap and which AWS script provides the correct value calculation?

3. The team wants to migrate from Secondary IP to Prefix Delegation without downtime. Describe the correct sequence of steps. Why is the rolling replace approach for existing nodes risky?

4. You want pods in the `databases` namespace to use EBS volumes encrypted with a specific CMK (not `aws/ebs`). What additional permissions does the EBS CSI Driver role need? Where should these permissions be configured (policy on the key? policy on the role?)?

5. A developer creates a PVC with `accessModes: ReadWriteMany` using the `gp3` StorageClass. What happens and why? What would be the correct CSI driver for `ReadWriteMany` storage in an EKS workload?

---

## References

- [FACT] [Amazon EKS add-ons](https://docs.aws.amazon.com/eks/latest/userguide/eks-add-ons.html) — docs.aws.amazon.com
- [FACT] [Assign more IP addresses to Amazon EKS nodes with prefixes](https://docs.aws.amazon.com/eks/latest/userguide/cni-increase-ip-addresses.html) — docs.aws.amazon.com
- [FACT] [Use Kubernetes volume storage with Amazon EBS](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html) — docs.aws.amazon.com
- [FACT] [create-addon — AWS CLI Reference](https://docs.aws.amazon.com/cli/latest/reference/eks/create-addon.html) — docs.aws.amazon.com
- [FACT] [Prefix Delegation mode for Linux — EKS Best Practices](https://docs.aws.amazon.com/eks/latest/best-practices/prefix-mode-linux.html) — docs.aws.amazon.com
- [FACT] [aws-ebs-csi-driver GitHub — AmazonEBSCSIDriverPolicyV2 migration](https://github.com/kubernetes-sigs/aws-ebs-csi-driver/issues/2918) — github.com
