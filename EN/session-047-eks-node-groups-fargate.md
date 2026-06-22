# Session 047 — EKS: Managed Node Groups vs Fargate, Upgrades, and Node Drain

**Estimated duration:** 60 min  
**Prerequisite:** session-046 (EKS cluster provisioning, eksctl, VPC CNI)

---

## Session Objectives

- Configure managed node groups with labels and taints to separate workloads by profile
- Create Fargate Profiles with selectors by namespace/labels and understand all their limitations
- Execute cluster version upgrade in the correct order (control plane → add-ons → node groups)
- Understand the 4 phases of managed node group update (Setup, Scale Up, Upgrade, Scale Down)
- Execute `kubectl drain` on a node without workload downtime

---

## 1. Managed Node Groups — Fundamental Concepts

[FACT] Every managed node group is implemented as an **EC2 Auto Scaling Group** managed by EKS, running within the customer's account. EKS provides the high-level API; the EC2 and ASG resources are in the customer's account and visible in the EC2 console.

[FACT] EKS automatically applies the following labels to each node in a managed node group (prefixed with `eks.amazonaws.com/`):

```
eks.amazonaws.com/nodegroup=<nodegroup-name>
eks.amazonaws.com/nodegroup-image=<ami-id>
eks.amazonaws.com/capacityType=ON_DEMAND  # or SPOT
```

[FACT] Additional labels and taints can be applied and **updated** via `update-nodegroup-config` without needing to recreate the node group. Label/taint updates are applied to **all new nodes**; existing nodes receive the update on the next rolling update (when the AMI is updated).

### 1.1 Workload separation with labels and taints

```
Strategy: multiple node groups per workload profile

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
│  │  (none)         │  │  tier=batch     │  │ taint:          │  │
│  │                 │  │  :NoSchedule   │  │  tier=ml        │  │
│  └─────────────────┘  └─────────────────┘  │  :NoSchedule   │  │
│                                             └─────────────────┘  │
│  App workload:          Batch workload:      ML workload:         │
│  nodeSelector:          nodeSelector:        nodeSelector:        │
│    tier: app              tier: batch          accelerator: gpu   │
│                         tolerations:          tolerations:        │
│                         - key: tier             - key: tier       │
│                           value: batch            value: ml       │
│                           effect: NoSchedule      effect: NOS..  │
└──────────────────────────────────────────────────────────────────┘
```

### 1.2 Capacity Types: On-Demand and Spot in node groups

[FACT] A managed node group can only be **ON_DEMAND or SPOT** — it is not possible to mix both types in the same node group. For workloads that use both, create two separate node groups (one OD and one Spot).

[FACT] For Spot, the allocation strategy is:
- `price-capacity-optimized` (PCO) — K8s clusters **1.28+** (most recent, recommended)
- `capacity-optimized` (CO) — K8s clusters **1.27 and earlier** (does not change for existing node groups)

[FACT] Spot managed node groups have **Capacity Rebalancing enabled** by default: when a Spot node receives a Rebalance Recommendation, EKS tries to launch a replacement node before draining the original.

---

## 2. Fargate Profiles — Selectors and Limitations

### 2.1 Fargate execution model

[FACT] Each Pod on Fargate has **its own dedicated VM** (isolated kernel, CPU, memory, and ENI). There is no shared host between Pods. This offers stronger isolation than EC2, but removes capabilities that depend on the host.

[FACT] On Fargate, **each Pod = one node** in `kubectl get nodes`. The node name follows the pattern `fargate-ip-<ip>.region.compute.internal`.

```bash
kubectl get nodes
# NAME                                                       STATUS  VERSION
# fargate-ip-10-0-1-20.us-east-1.compute.internal           Ready   v1.33.0-eks-xxx
# fargate-ip-10-0-1-21.us-east-1.compute.internal           Ready   v1.33.0-eks-xxx
# ip-10-0-2-10.us-east-1.compute.internal                   Ready   v1.33.0-eks-xxx  ← EC2 node
```

### 2.2 Fargate limitations (complete table)

[FACT] The following limitations apply to Fargate on EKS:

```
╔═══════════════════════════════════════════════╦═══════╦═════════════════════════════╗
║ Resource / Capability                         ║ Suprt ║ Alternative                 ║
╠═══════════════════════════════════════════════╬═══════╬═════════════════════════════╣
║ DaemonSets                                    ║  NO   ║ Sidecar container           ║
║ Privileged containers                         ║  NO   ║ Use EC2 node group          ║
║ HostPort / HostNetwork                        ║  NO   ║ Service ClusterIP or LB     ║
║ GPUs                                          ║  NO   ║ EC2 node group (p3, g4dn)   ║
║ Amazon EBS (volumes)                          ║  NO   ║ Amazon EFS (auto-mount)     ║
║ Public subnets                                ║  NO   ║ Private subnet + NAT GW     ║
║ Spot capacity type                            ║  NO   ║ Managed node group Spot     ║
║ Custom AMI                                    ║  NO   ║ EC2 node group              ║
║ Custom CNI                                    ║  NO   ║ EC2 node group              ║
║ SSH access                                    ║  NO   ║ kubectl exec                ║
║ AWS Outposts / Wavelength / Local Zones       ║  NO   ║ EC2 node group              ║
║ Windows containers                            ║  NO   ║ EC2 Windows node group      ║
║ ARM / Graviton                                ║  NO   ║ EC2 Graviton node group     ║
║ IMDS (instance metadata)                      ║  NO   ║ IRSA for IAM credentials    ║
╠═══════════════════════════════════════════════╬═══════╬═════════════════════════════╣
║ Amazon EFS (static provisioning)              ║  YES  ║ —                           ║
║ ALB / NLB (target type IP)                    ║  YES  ║ — (does not use node IP mode)║
║ Security Groups per Pod (via IRSA/SG for Pods)║  YES  ║ —                           ║
║ VPA / HPA                                     ║  YES  ║ —                           ║
╚═══════════════════════════════════════════════╩═══════╩═════════════════════════════╝
```

[FACT] **IMDS is not available** for Pods on Fargate. Applications that need IAM credentials **must use IRSA** (IAM Roles for Service Accounts). Applications that access IMDS to obtain Region or AZ must have those values hard-coded in the Pod spec or via `downwardAPI`.

### 2.3 Fargate Profile — structure

[FACT] Each Fargate Profile can have up to **5 selectors**. Each selector requires namespace (mandatory field) and can have optional labels. A Pod must match at least one selector to be scheduled on Fargate.

```yaml
# Fargate profile via eksctl ClusterConfig
fargateProfiles:
  - name: fp-apps
    selectors:
      # Selector 1: any pod in "analytics" namespace with label fargate=true
      - namespace: analytics
        labels:
          fargate: "true"
      # Selector 2: any pod in "batch" namespace (no label filter)
      - namespace: batch
      # Selector 3: CoreDNS pods in kube-system
      - namespace: kube-system
        labels:
          k8s-app: kube-dns
    # Private subnets for Fargate pods (required)
    subnets:
      - subnet-0111aaaa
      - subnet-0222bbbb
      - subnet-0333cccc
```

[FACT] Pods that do not match any Fargate profile remain in **Pending** state indefinitely (they are not scheduled on Fargate or EC2 automatically). If there is an EC2 node group in the cluster, the scheduler will try to place the Pod there.

### 2.4 Fargate Pod Sizing

[FACT] Fargate provisions a micro-VM with resources equal to the sum of requests from all containers in the Pod, rounded up to the next supported combination. Available combinations: 0.25–16 vCPU and 0.5–120 GB (with minimum ratio rules).

[FACT] Use **Vertical Pod Autoscaler (VPA)** with mode `Auto` or `Recreate` to adjust Fargate pod sizing — VPA recreates the Pod with new resource requests, and Fargate provisions the corresponding VM.

---

## 3. Cluster Version Upgrade — Order and Restrictions

### 3.1 Fundamental rules

[FACT] The EKS cluster upgrade must follow the **mandatory order**:

```
1. Control Plane  →  2. Add-ons  →  3. Node Groups  →  4. kubectl
    (EKS manages)      (VPC CNI,       (managed NG         (1 minor
                       CoreDNS,        rolling update)      version skew)
                       kube-proxy,
                       EBS CSI)
```

[FACT] The upgrade is **only one minor version at a time** — it is not possible to jump from 1.32 directly to 1.34. Each step (1.32→1.33, 1.33→1.34) requires a separate upgrade.

[FACT] The control plane upgrade is **irreversible** — it is not possible to downgrade. If you need to go back to a previous version, you must create a new cluster at the desired version and migrate workloads.

[FACT] Starting from K8s 1.28, the kubelet can be up to **3 minor versions** behind the kube-apiserver (upstream skew policy). In practice, with kubelet 1.30 you can upgrade the control plane to 1.31, 1.32, and 1.33 before updating the node group.

[FACT] EKS requires up to **5 available IPs** in the cluster subnets to create new ENIs during the control plane upgrade. If the subnet is full, the upgrade may fail.

### 3.2 Complete upgrade flow

```
┌─────────────────────────────────────────────────────────────────────┐
│ Upgrade 1.32 → 1.33                                                  │
│                                                                      │
│ Step 1: Verify cluster state                                         │
│   aws eks list-insights --cluster-name checkout-prod                 │
│   kubectl get nodes (all should be Ready + current version)          │
│                                                                      │
│ Step 2: Upgrade Control Plane                                        │
│   aws eks update-cluster-version --name checkout-prod                │
│   --kubernetes-version 1.33                                          │
│   [~15 min — rolling update of API servers, no downtime]             │
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
│   (existing Fargate pods are not auto-updated)                       │
│                                                                      │
│ Step 6: Update kubectl                                               │
│   brew upgrade kubectl  # or equivalent                              │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.3 Managed node group update phases

[FACT] When EKS updates a managed node group (AMI update or K8s version update), it executes 4 phases:

```
┌─────────────────────────────────────────────────────────────────────┐
│ Phase 1: SETUP                                                       │
│   • Creates new version of EC2 Launch Template for the ASG           │
│   • Updates ASG to use the new LT version                            │
│   • Determines maxUnavailable (default: 1 node, maximum: 100)       │
│                                                                      │
│ Phase 2: SCALE UP                                                    │
│   • Increases max and desired count of the ASG                       │
│   • Launches new nodes (new AMI/version) in the same AZs            │
│   • Waits for new nodes to become Ready with EKS labels             │
│   • Cordons + labels "exclude-from-external-load-balancers" on old  │
│   • Timeout: 15 min per node to boot and join the cluster           │
│                                                                      │
│ Phase 3: UPGRADE (default strategy)                                  │
│   • Selects old node randomly (respects maxUnavailable)             │
│   • Drains pods from node (eviction — respects PDB)                 │
│   • Waits 60 seconds after cordon                                    │
│   • Sends termination to ASG                                         │
│   • Repeats until all nodes use new LT version                       │
│   • PodEvictionFailure → upgrade fails (if 15 min timeout)          │
│                                                                      │
│ Phase 4: SCALE DOWN                                                  │
│   • Returns max and desired count of the ASG to original values     │
│   • If Cluster Autoscaler is scaling during this step,               │
│     the workflow exits immediately                                    │
└─────────────────────────────────────────────────────────────────────┘
```

[FACT] There are two **update strategies** for Phase 3:
- **Default** (recommended): launches new nodes first, then terminates old ones — total capacity never drops below configured value
- **Minimal**: terminates old nodes first, then launches new ones — capacity drops temporarily; suitable for GPU (avoid paying for two GPU sets simultaneously)

[FACT] During a version update, EKS **respects Pod Disruption Budgets (PDB)**. However PDBs **are not respected** in AZRebalance operations or desired count reduction — those actions attempt eviction for up to 15 minutes and terminate the node regardless.

---

## 4. kubectl drain — Zero-Downtime Node Maintenance

### 4.1 Difference between cordon and drain

```
kubectl cordon <node>
  → Marks the node as Unschedulable
  → EXISTING Pods continue running
  → No NEW pods are scheduled on the node
  → Used when you need time to prepare the drain

kubectl drain <node>
  → Automatically cordons
  → EVICTS all pods (except DaemonSets if --ignore-daemonsets)
  → Waits for each pod to be rescheduled on another node
  → Indicates the node is ready for maintenance/shutdown
```

### 4.2 Essential drain flags

[FACT] Mandatory flags for drain in real clusters:

```bash
kubectl drain <node-name> \
  --ignore-daemonsets \          # mandatory: DaemonSet pods cannot be evicted
  --delete-emptydir-data \       # mandatory: pods with emptyDir (local) would be blocked
  --grace-period=60 \            # override of terminationGracePeriodSeconds (60s default)
  --timeout=300s \               # maximum total time for the drain to complete
  --force                        # forces removal of pods without controller (unmanaged)
```

[FACT] If a Pod has a **PDB that blocks eviction** (e.g.: `minAvailable=1` and there is only 1 replica), the drain waits indefinitely (or until `--timeout`). Solution: temporarily increase replicas, or adjust the PDB.

### 4.3 PodDisruptionBudget — prerequisite for safe drain

```yaml
# PDB to ensure at least 1 pod is always available during maintenance
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: checkout-api-pdb
  namespace: production
spec:
  # minAvailable: minimum pods that must be Up during disruption
  minAvailable: 1
  # OR maxUnavailable: maximum pods that can be Down during disruption
  # maxUnavailable: 1
  selector:
    matchLabels:
      app: checkout-api
```

[FACT] `minAvailable` and `maxUnavailable` can be absolute numbers or percentages (e.g.: `50%`). With `minAvailable: 1` and 3 replicas, drain can evict up to 2 pods at a time (2 maxUnavailable).

---

## 5. CDK Python — Node Groups, Fargate Profile, Taints, and Labels

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
        # Node group 1: Application — On-Demand, no taint (main workload)
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
            # No taint — accepts any pod that doesn't have nodeSelector/affinity
            tags={"project": "checkout", "env": "prod"},
        )

        # ──────────────────────────────────────────────────────────────
        # Node group 2: Batch/Jobs — Spot, with NoSchedule taint
        # Only pods with toleration tier=batch:NoSchedule are scheduled here
        # ──────────────────────────────────────────────────────────────
        ng_batch = cluster.add_nodegroup_capacity("NgBatch",
            nodegroup_name="ng-batch-spot",
            instance_types=[
                ec2.InstanceType("c5.xlarge"),
                ec2.InstanceType("c5a.xlarge"),
                ec2.InstanceType("c5n.xlarge"),
                ec2.InstanceType("c4.xlarge"),
                ec2.InstanceType("m5.xlarge"),   # diversify Spot pools
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
                    effect=eks.TaintEffect.NO_SCHEDULE,  # prevents pods without toleration
                )
            ],
        )

        # ──────────────────────────────────────────────────────────────
        # Node group 3: Infra — On-Demand, PreferNoSchedule taint
        # Monitoring, logging, cluster-autoscaler etc.
        # PreferNoSchedule: pods WITHOUT toleration prefer other nodes but can use this one
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
        # Fargate Profile — analytics and batch namespaces
        # ──────────────────────────────────────────────────────────────
        fargate_profile = cluster.add_fargate_profile("FargateAnalytics",
            fargate_profile_name="fp-analytics",
            selectors=[
                # Analytics: only pods with label fargate=true
                eks.Selector(
                    namespace="analytics",
                    labels={"fargate": "true"},
                ),
                # Batch jobs: any pod in batch-fargate namespace
                eks.Selector(
                    namespace="batch-fargate",
                ),
            ],
            # Fargate requires private subnets
            subnet_selection=ec2.SubnetSelection(
                subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS,
            ),
        )


class EksFargateWorkloadStack(Stack):
    """
    Examples of Kubernetes manifests applied via CDK (eks.KubernetesManifest).
    """
    def __init__(self, scope: Construct, construct_id: str,
                 cluster: eks.Cluster, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # ──────────────────────────────────────────────────────────────
        # PodDisruptionBudget for the API workload
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
        # API Deployment — nodeSelector for ng-app, no batch toleration
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
                        # Ensures scheduling only on nodes with tier=app
                        "nodeSelector": {"tier": "app"},
                        # topologySpreadConstraints: distributes replicas across AZs
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
                            # default terminationGracePeriodSeconds is 30s
                        }],
                        "terminationGracePeriodSeconds": 60,
                    },
                },
            },
        })

        # ──────────────────────────────────────────────────────────────
        # Batch Job on Spot node group — with tier=batch toleration
        # ──────────────────────────────────────────────────────────────
        cluster.add_manifest("BatchJob", {
            "apiVersion": "batch/v1",
            "kind": "Job",
            "metadata": {"name": "report-generator", "namespace": "batch"},
            "spec": {
                "ttlSecondsAfterFinished": 3600,   # cleans up job 1h after completion
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
        # Pod on Fargate — analytics namespace with label fargate=true
        # IMDS unavailable: no hardcoded AWS_METADATA_SERVICE_TIMEOUT
        # Credentials via IRSA (serviceAccountName + annotation)
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
                        # serviceAccountName with IRSA annotation for IAM credentials
                        "serviceAccountName": "analytics-sa",
                        "containers": [{
                            "name": "processor",
                            "image": "my-ecr.dkr.ecr.us-east-1.amazonaws.com/analytics:v2",
                            "resources": {
                                # Fargate rounds up to supported combination
                                "requests": {"cpu": "1000m", "memory": "2Gi"},
                                "limits":   {"cpu": "1000m", "memory": "2Gi"},
                            },
                            "env": [
                                # AWS_REGION hard-coded — IMDS unavailable on Fargate
                                {"name": "AWS_REGION", "value": "us-east-1"},
                                {"name": "AWS_DEFAULT_REGION", "value": "us-east-1"},
                            ],
                        }],
                        # Fargate ignores nodeSelector — pod is scheduled via profile
                    },
                },
            },
        })
```

---

## 6. Python — Upgrade Orchestration Script

```python
"""
EKS cluster upgrade script.
Validates prerequisites and executes upgrade in correct order.
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
    """Validates prerequisites for the upgrade. Returns list of errors."""
    errors = []

    # 1. Check cluster status
    if state.status != "ACTIVE":
        errors.append(f"Cluster is not ACTIVE: {state.status}")

    # 2. Check version skew (only 1 minor version)
    curr_parts = [int(x) for x in state.current_version.split(".")]
    tgt_parts  = [int(x) for x in state.target_version.split(".")]
    if tgt_parts[1] - curr_parts[1] != 1:
        errors.append(
            f"Upgrade must be 1 minor version at a time: {state.current_version} → {state.target_version} "
            f"(difference: {tgt_parts[1] - curr_parts[1]} minor versions)"
        )

    # 3. Check node groups — all must be at current control plane version
    for ng in state.node_groups:
        if ng["version"] != state.current_version:
            errors.append(
                f"Node group {ng['name']} is at version {ng['version']}, "
                f"but control plane is at {state.current_version}. "
                "Update the node group before upgrading the control plane."
            )
        if ng["status"] != "ACTIVE":
            errors.append(f"Node group {ng['name']} is not ACTIVE: {ng['status']}")

    return errors


def wait_for_cluster_active(cluster_name: str, timeout_min: int = 30) -> bool:
    """Waits for cluster to become ACTIVE. Returns True on success."""
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
    """Initiates control plane upgrade. Returns update_id."""
    print(f"\n[Step 2] Upgrading control plane {state.current_version} → {state.target_version}...")
    response = eks.update_cluster_version(
        name=state.name,
        kubernetesVersion=state.target_version,
    )
    update_id = response["update"]["id"]
    print(f"  Update started: {update_id}")
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
            print(f"  Update failed: {resp['update'].get('errors', [])}")
            return False
        time.sleep(30)
    return False


def upgrade_addon(cluster_name: str, addon_name: str) -> bool:
    """Updates an add-on to the latest compatible version with the cluster."""
    try:
        # Get current cluster version
        cluster_version = eks.describe_cluster(name=cluster_name)["cluster"]["version"]

        # List compatible versions and choose the latest
        versions_resp = eks.describe_addon_versions(
            kubernetesVersion=cluster_version,
            addonName=addon_name,
        )
        latest = versions_resp["addons"][0]["addonVersions"][0]["addonVersion"]

        print(f"  Updating add-on {addon_name} → {latest}")
        eks.update_addon(
            clusterName=cluster_name,
            addonName=addon_name,
            addonVersion=latest,
            resolveConflicts="OVERWRITE",
        )
        # Wait for add-on to become ACTIVE
        for _ in range(20):
            status = eks.describe_addon(clusterName=cluster_name, addonName=addon_name)
            if status["addon"]["status"] == "ACTIVE":
                return True
            time.sleep(15)
        return False
    except Exception as e:
        print(f"  Error updating add-on {addon_name}: {e}")
        return False


def upgrade_nodegroup(cluster_name: str, ng_name: str, target_version: str) -> bool:
    """Initiates rolling update of the node group."""
    print(f"\n  Upgrading node group {ng_name} → {target_version}...")
    try:
        resp = eks.update_nodegroup_version(
            clusterName=cluster_name,
            nodegroupName=ng_name,
            version=target_version,
        )
        update_id = resp["update"]["id"]
        # Wait for completion (can take 30-60 min for large node groups)
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
        print(f"  Error updating node group {ng_name}: {e}")
        return False


def run_cluster_upgrade(cluster_name: str, target_version: str) -> None:
    print(f"=== EKS Cluster Upgrade: {cluster_name} → {target_version} ===\n")

    # Step 1: Validate prerequisites
    print("[Step 1] Validating prerequisites...")
    state = get_cluster_state(cluster_name, target_version)
    errors = validate_upgrade_prerequisites(state)
    if errors:
        print("  ERRORS FOUND — upgrade aborted:")
        for err in errors:
            print(f"  ✗ {err}")
        sys.exit(1)
    print("  ✓ Prerequisites OK")

    # Step 2: Upgrade control plane
    update_id = upgrade_control_plane(state)
    if not wait_for_cluster_update(cluster_name, update_id):
        print("  ✗ Control plane upgrade failed")
        sys.exit(1)
    print("  ✓ Control plane updated")

    # Step 3: Upgrade add-ons
    print("\n[Step 3] Updating add-ons...")
    for addon_name in ["vpc-cni", "coredns", "kube-proxy", "aws-ebs-csi-driver"]:
        if not upgrade_addon(cluster_name, addon_name):
            print(f"  ✗ Failed to update add-on {addon_name}")

    # Step 4: Upgrade node groups
    print("\n[Step 4] Updating node groups...")
    state = get_cluster_state(cluster_name, target_version)  # re-read current state
    for ng in state.node_groups:
        if ng["version"] != target_version:
            if not upgrade_nodegroup(cluster_name, ng["name"], target_version):
                print(f"  ✗ Failed to update node group {ng['name']}")
            else:
                print(f"  ✓ Node group {ng['name']} updated")

    print("\n=== Upgrade complete ===")
    print(f"Next step: kubectl rollout restart (Fargate pods are not auto-updated)")
```

---

## 7. CLI — Essential Examples

```bash
# ─────────────────────────────────────────────────────────────
# MANAGED NODE GROUPS — labels and taints
# ─────────────────────────────────────────────────────────────

# Add/update labels and taints on node group (without recreating nodes)
aws eks update-nodegroup-config \
  --cluster-name checkout-prod \
  --nodegroup-name ng-batch-spot \
  --labels 'addOrUpdateLabels={tier=batch,updated-at=2026-06-13}' \
  --taints 'addOrUpdateTaints=[{key=tier,value=batch,effect=NO_SCHEDULE}]'

# Remove label from a node group
aws eks update-nodegroup-config \
  --cluster-name checkout-prod \
  --nodegroup-name ng-batch-spot \
  --labels 'removeLabels=[updated-at]'

# Remove taint from a node group
aws eks update-nodegroup-config \
  --cluster-name checkout-prod \
  --nodegroup-name ng-batch-spot \
  --taints 'removeTaints=[{key=tier,effect=NO_SCHEDULE}]'

# List all node groups with version and status
aws eks list-nodegroups --cluster-name checkout-prod
aws eks describe-nodegroup \
  --cluster-name checkout-prod \
  --nodegroup-name ng-app-ondemand \
  --query 'nodegroup.{Name:nodegroupName,Version:version,Status:status,Capacity:capacityType,Labels:labels,Taints:taints}'

# ─────────────────────────────────────────────────────────────
# FARGATE PROFILES
# ─────────────────────────────────────────────────────────────

# Create Fargate profile via CLI
aws eks create-fargate-profile \
  --cluster-name checkout-prod \
  --fargate-profile-name fp-analytics \
  --pod-execution-role-arn arn:aws:iam::123456789012:role/EKSFargatePodExecutionRole \
  --subnets subnet-0111aaaa subnet-0222bbbb subnet-0333cccc \
  --selectors 'namespace=analytics,labels={fargate=true}' \
              'namespace=batch-fargate'

# List Fargate profiles
aws eks list-fargate-profiles --cluster-name checkout-prod

# Check status and selectors of a profile
aws eks describe-fargate-profile \
  --cluster-name checkout-prod \
  --fargate-profile-name fp-analytics \
  --query 'fargateProfile.{Status:status,Selectors:selectors}'

# ─────────────────────────────────────────────────────────────
# CLUSTER UPGRADE
# ─────────────────────────────────────────────────────────────

# Step 0: Check upgrade insights (deprecated APIs, compatibility issues)
aws eks list-insights \
  --cluster-name checkout-prod \
  --filter '{"categories":["UPGRADE_READINESS"]}' \
  --query 'insights[*].{Name:name,Status:insightStatus.status,Recommendation:recommendation}'

# Step 1: Check current version and nodes
kubectl version
kubectl get nodes -o wide

# Step 2: Upgrade control plane (wait ~15 min)
aws eks update-cluster-version \
  --name checkout-prod \
  --kubernetes-version 1.33 \
  --region us-east-1

# Monitor progress
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

# Step 4: Upgrade node groups (one at a time, wait for completion)
aws eks update-nodegroup-version \
  --cluster-name checkout-prod \
  --nodegroup-name ng-app-ondemand \
  --kubernetes-version 1.33   # or without --kubernetes-version to only update AMI

# Monitor node group update
aws eks describe-update \
  --name checkout-prod \
  --nodegroup-name ng-app-ondemand \
  --update-id <update-id> \
  --query 'update.{Status:status,Type:type,Errors:errors}'

# Step 5: Redeploy Fargate pods (not auto-updated)
kubectl rollout restart deployment/analytics-processor -n analytics

# ─────────────────────────────────────────────────────────────
# kubectl drain — node maintenance
# ─────────────────────────────────────────────────────────────

# List nodes and identify which one to drain
kubectl get nodes -o wide

# Cordon (prevents new pods without evicting existing ones)
kubectl cordon ip-10-0-2-10.us-east-1.compute.internal

# Check pods on the node
kubectl get pods -A -o wide --field-selector spec.nodeName=ip-10-0-2-10.us-east-1.compute.internal

# Drain (cordon + evict)
kubectl drain ip-10-0-2-10.us-east-1.compute.internal \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=60 \
  --timeout=300s

# Verify all pods were moved
kubectl get pods -A -o wide | grep ip-10-0-2-10

# After maintenance: uncordon to accept pods again
kubectl uncordon ip-10-0-2-10.us-east-1.compute.internal

# Check PDBs that may block drain
kubectl get pdb -A
kubectl describe pdb checkout-api-pdb -n production
```

---

## 8. Pitfalls

[FACT] **Fargate ignores nodeSelector** — pods scheduled on Fargate do not respect user-defined `nodeSelector`. The scheduling criterion is exclusively the Fargate Profile (namespace + labels). If a pod does not match any profile, it stays Pending.

[FACT] **Existing DaemonSets in clusters with Fargate** cause Pending pods: the Fargate scheduler tries to create a DaemonSet pod on the "Fargate node", but DaemonSets are not supported. Solution: add `nodeAffinity` to the DaemonSet to run only on EC2 nodes (via label `eks.amazonaws.com/compute-type: ec2`).

[FACT] **Control plane upgrade can fail silently if subnets are full**: the error `InsufficientFreeAddressesInSubnet` appears in describe-update but the cluster remains at the previous version (automatic rollback). Check available IPs in cluster subnets before upgrading.

[FACT] **Spot node groups with `maxUnavailable=1` and many AZs can launch many nodes temporarily**: scale up launches up to `2 × number of AZs` nodes (e.g.: 3 AZs = 6 extra nodes before draining old ones). This may trigger EC2 instance quota limits.

[CONSENSUS] **PDB with `minAvailable=N` where N = number of replicas blocks drain indefinitely**: if the deployment has 2 replicas and the PDB requires `minAvailable=2`, no pod can be evicted. Always configure PDB with slack (`minAvailable = total_replicas - 1` or `maxUnavailable >= 1`).

[FACT] **Fargate jobs need `ttlSecondsAfterFinished`**: completed Job pods on Fargate continue generating charges (pod CPU/memory) after completion. Always configure TTL to avoid zombie pod costs.

---

## Reflection Exercise

You need to redesign the compute topology of an EKS cluster with 4 workload types:

1. **API Gateway** (stateless, high traffic, SLA 99.9%) — 10 replicas, zero tolerance for interruption
2. **ML Inference** (stateless, CPU-intensive, tolerant to 10% interruptions) — 50 replicas
3. **Batch ETL** (short jobs, 5-30 min, highly tolerant to interruptions) — 0–200 pods variable
4. **Monitoring Stack** (Prometheus, Grafana — stateful, DaemonSets needed) — 3 replicas

**Answer:**

1. How many managed node groups are needed? Which capacity type (OD/Spot) for each? Which labels and taints?
2. Should ML Inference go to Fargate or EC2 node group? Why? (consider: DaemonSets for metrics, pod volume, cost)
3. Can the Monitoring Stack go to Fargate? Why?
4. How to configure PDBs to ensure `kubectl drain` works without blocking for more than 5 minutes for each workload?
5. During cluster upgrade 1.32→1.33, in what exact order should the 4 node groups be updated? Is there any order dependency?

---

## References

- [FACT] [Simplify node lifecycle with managed node groups](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html) — docs.aws.amazon.com
- [FACT] [Understand each phase of node updates](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-update-behavior.html) — docs.aws.amazon.com
- [FACT] [Simplify compute management with AWS Fargate](https://docs.aws.amazon.com/eks/latest/userguide/fargate.html) — docs.aws.amazon.com
- [FACT] [Define which Pods use AWS Fargate when launched](https://docs.aws.amazon.com/eks/latest/userguide/fargate-profile.html) — docs.aws.amazon.com
- [FACT] [Update existing cluster to new Kubernetes version](https://docs.aws.amazon.com/eks/latest/userguide/update-cluster.html) — docs.aws.amazon.com
- [FACT] [Recipe: Prevent pods from being scheduled on specific nodes (taints)](https://docs.aws.amazon.com/eks/latest/userguide/node-taints-managed-node-groups.html) — docs.aws.amazon.com
