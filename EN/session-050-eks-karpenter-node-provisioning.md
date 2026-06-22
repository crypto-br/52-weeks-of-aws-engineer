# Session 050 — EKS: Karpenter — Dynamic Node Provisioning and NodePools

**Estimated duration:** 60 min  
**Prerequisite:** session-049 (EKS add-ons, VPC CNI, EBS CSI)

---

## Session Objectives

- Understand Karpenter's architecture and how it differs from Cluster Autoscaler
- Install Karpenter via Helm with the correct IAM policies
- Create NodePool and EC2NodeClass with instance constraints, taint/toleration, and disruption budgets
- Verify that Karpenter provisions and consolidates nodes in response to Pending/removed pods
- Decide when to use Karpenter vs Cluster Autoscaler vs managed node groups

---

## 1. Karpenter vs Cluster Autoscaler — Comparison

### 1.1 Mental model

[FACT] The Cluster Autoscaler (CAS) operates at the **Auto Scaling Groups (ASGs)** level: when the Kubernetes scheduler cannot place a pod, CAS checks which ASGs could absorb the pod and increases the group's `desiredCapacity`. Karpenter operates at the **pods directly** level: when the scheduler cannot allocate a pod, Karpenter directly calls the EC2 API to create the instance that best meets the requested resources.

```
┌─────────────────────────────────────────────────────────────────────┐
│ Cluster Autoscaler                                                  │
│                                                                     │
│  Pod Pending                                                        │
│       │                                                             │
│       ▼                                                             │
│  CAS checks pre-configured node groups                              │
│  (each node group = 1 instance type or restricted family)           │
│       │                                                             │
│       ▼                                                             │
│  Increases desiredCapacity of the most suitable ASG                  │
│       │                                                             │
│       ▼                                                             │
│  EC2 Auto Scaling creates instance (can take 2-5 min)               │
│       │                                                             │
│       ▼                                                             │
│  Node registers in the cluster → pod is scheduled                   │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│ Karpenter                                                           │
│                                                                     │
│  Pod Pending (watch via K8s API)                                    │
│       │                                                             │
│       ▼                                                             │
│  Karpenter reads pod requirements (resources, nodeSelector,         │
│  affinity, tolerations, topologySpread)                             │
│       │                                                             │
│       ▼                                                             │
│  Selects the most cost-effective instance that meets requirements   │
│  of all pending pods simultaneously (bin-packing)                   │
│       │                                                             │
│       ▼                                                             │
│  Calls EC2 Fleet API directly (RunInstances / CreateFleet)          │
│  Creates NodeClaim CRD to track state                               │
│       │                                                             │
│       ▼                                                             │
│  Node registers → pod scheduled (usually in < 60s)                  │
└─────────────────────────────────────────────────────────────────────┘
```

[FACT] Structural comparison:

```
╔══════════════════════════╦═══════════════════════════╦═══════════════════════════╗
║ Dimension                ║ Cluster Autoscaler (CAS)  ║ Karpenter                 ║
╠══════════════════════════╬═══════════════════════════╬═══════════════════════════╣
║ Unit of scale            ║ Node Group (ASG)          ║ Individual EC2 instance   ║
║ Instance decision        ║ Pre-configured in NG      ║ Runtime (best fit)        ║
║ No. of node groups       ║ Many (1 per workload)     ║ Few (1-3 NodePools)       ║
║ Scale-up speed           ║ 2-5 min (ASG trigger)     ║ < 60s (direct EC2 API)    ║
║ Consolidation (scale-down)║ 10 min idle default      ║ Configurable (1m+)        ║
║ K8s versioning           ║ Coupled (version-specific)║ Decoupled                 ║
║ Spot diversity           ║ Manual (1 NG per family)  ║ Automatic (broad pool)    ║
║ Bin-packing              ║ Limited (per ASG)         ║ Global (all pods)         ║
║ AWS API                  ║ Auto Scaling API          ║ EC2 Fleet / RunInstances  ║
╚══════════════════════════╩═══════════════════════════╩═══════════════════════════╝
```

[CONSENSUS] **When to prefer Karpenter**: clusters with variable/spiky demand, heterogeneous workloads, need for Spot with high instance diversity, or when the overhead of maintaining dozens of node groups is excessive.

[CONSENSUS] **When to prefer CAS or static node groups**: stable and predictable workloads, when organizational constraints prevent IAM with broad RunInstances/TerminateInstances powers, or clusters requiring compliance with very specific node configurations.

---

## 2. Karpenter Architecture

### 2.1 Components

[FACT] Karpenter runs as a **Deployment** with 2 replicas (controller + webhook) in `kube-system`. It is **not** a managed EKS add-on — it is installed via Helm chart from the OCI registry `public.ecr.aws/karpenter/karpenter`.

[FACT] CRDs created by Karpenter:

```
karpenter.sh/v1:
  NodePool      — scheduling constraints and disruption policies
  NodeClaim     — represents an EC2 instance being provisioned/active

karpenter.k8s.aws/v1:
  EC2NodeClass  — AWS-specific configuration (AMI, subnet, SG, role)

karpenter.sh/v1 (readonly):
  NodeOverlay   — configuration overlay on existing EC2NodeClass
```

[FACT] Karpenter must run on a node **not managed by itself** — on a managed node group or on Fargate. If the only node in the cluster is provisioned by Karpenter and it needs to be removed (consolidation), the Karpenter controller would have nowhere to run.

### 2.2 Provisioning flow

```
1. Pod becomes Pending (scheduler cannot find a suitable node)
2. Karpenter detects the pod via watch on K8s API
3. Karpenter groups pending pods that can be co-located
4. Selects appropriate EC2NodeClass and NodePool
5. Chooses optimal instance (bin-packing + cost + availability)
6. Creates NodeClaim CRD (tracks state)
7. Calls EC2 Fleet API to create the instance
8. Instance bootstrapping: nodeadm/userdata configures the kubelet
9. Node appears in kubectl get nodes
10. Karpenter associates the NodeClaim with the Node
11. Scheduler schedules pods on the new node
```

### 2.3 IAM — Karpenter Policies

[FACT] The official installation CloudFormation creates 6 separate IAM policies (v1.13):

```
KarpenterControllerNodeLifecyclePolicy     → RunInstances, TerminateInstances,
                                             CreateFleet, CreateLaunchTemplate,
                                             DeleteLaunchTemplate, ...
KarpenterControllerIAMIntegrationPolicy    → iam:PassRole (for KarpenterNodeRole),
                                             iam:AddRoleToInstanceProfile,
                                             iam:CreateInstanceProfile, ...
KarpenterControllerEKSIntegrationPolicy    → eks:DescribeCluster
KarpenterControllerInterruptionPolicy      → sqs:ReceiveMessage, sqs:DeleteMessage,
                                             sqs:GetQueueUrl,
                                             events:CreateEventBus, ...
KarpenterControllerResourceDiscoveryPolicy → ec2:Describe* (instances, AZs, subnets,
                                             SGs), pricing:GetProducts
KarpenterControllerZonalShiftPolicy        → arc-zonal-shift:GetManagedResource
```

[FACT] The `KarpenterNodeRole-<cluster>` is the IAM Role assigned to EC2 nodes created by Karpenter. It must have: `AmazonEKSWorkerNodePolicy`, `AmazonEKS_CNI_Policy`, `AmazonEC2ContainerRegistryReadOnly`, `AmazonSSMManagedInstanceCore`.

[FACT] **Tag security risk**: Karpenter uses 3 tags to associate EC2 instances with NodeClaims:
- `karpenter.sh/managed-by: <cluster-name>`
- `karpenter.sh/nodepool: <nodepool-name>`
- `kubernetes.io/cluster/<cluster-name>: owned`

Any user with `ec2:CreateTags`/`ec2:DeleteTags` on these tags for `i-*` instances can manipulate Karpenter. The recommendation is to use tag-based IAM policies to restrict `CreateTags`/`DeleteTags` to only the Karpenter role.

---

## 3. NodePool and EC2NodeClass — Complete Anatomy

### 3.1 NodePool

[FACT] The NodePool defines the constraints on the nodes that Karpenter can create. Each Pending pod is compared with the available NodePools and scheduled on the NodePool that best fits.

```yaml
# NodePool annotated with all relevant fields
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: general-compute
spec:
  # ── Template: configuration of nodes that will be created ─────────
  template:
    metadata:
      labels:
        team: platform        # labels propagated to the K8s Node
      annotations:
        example.com/owner: platform-team
    spec:
      # Reference to EC2NodeClass (AWS-specific configuration)
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default

      # Taints on the node — pods need to tolerate to be scheduled here
      taints: []

      # startupTaints: applied to the node, but pods do NOT need to tolerate.
      # Used to wait for initialization (e.g., Cilium CNI agent).
      # A DaemonSet or external controller must remove the taint.
      startupTaints: []

      # Node expiration (TTL): after 720h, the node is drained and terminated.
      # Useful to force rotation and apply OS/K8s patches.
      # 'Never' disables expiration.
      expireAfter: 720h

      # Maximum drain time before forcing termination
      terminationGracePeriod: 48h

      # Requirements: scheduling constraints (intersection with pod spec)
      # Operators: In, NotIn, Exists, DoesNotExist, Gt, Lt, Gte, Lte
      requirements:
        # Architecture
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]

        # OS
        - key: kubernetes.io/os
          operator: In
          values: ["linux"]

        # Capacity type
        # Automatic priority: reserved > spot > on-demand
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]

        # Instance categories (c=compute, m=general, r=memory)
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["c", "m", "r"]
          # minValues: requires at least N distinct categories in the pool
          # (avoids overfitting to a single family for Spot)
          minValues: 2

        # Minimum generation (avoids old instances)
        - key: karpenter.k8s.aws/instance-generation
          operator: Gte
          values: ["3"]

        # Exclude bare-metal instances (generally not needed)
        - key: karpenter.k8s.aws/instance-hypervisor
          operator: In
          values: ["nitro"]

  # ── Disruption: consolidation and rotation control ────────────────
  disruption:
    # WhenEmptyOrUnderutilized: consolidates empty AND underutilized nodes
    # WhenEmpty: consolidates only nodes without workload pods
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 1m    # waits 1 min of inactivity before consolidating

    # Budgets: limits how many nodes can be disrupted simultaneously
    budgets:
      - nodes: "10%"        # maximum 10% of nodes disrupted at once
      # During business hours (Mon-Fri 9am-5pm): no disruption
      - schedule: "0 9 * * mon-fri"
        duration: 8h
        nodes: "0"

  # ── Limits: resource ceiling this NodePool can consume ────────────
  limits:
    cpu: "1000"       # 1000 total vCPUs
    memory: 1000Gi    # 1 TiB total memory
    # nodes: 50       # optional: maximum nodes

  # ── Weight: priority when multiple NodePools are candidates ───────
  weight: 10
```

### 3.2 EC2NodeClass

[FACT] The EC2NodeClass contains all AWS-specific configuration. Multiple NodePools can reference the same EC2NodeClass.

```yaml
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  # IAM role for EC2 nodes (must exist with worker node policies)
  role: "KarpenterNodeRole-checkout-prod"

  # AMI: 'alias' allows using the latest optimized EKS AMI
  # Formats: al2023@latest, al2023@v20240101, al2@latest, bottlerocket@latest
  amiSelectorTerms:
    - alias: "al2023@latest"

  # Subnets: Karpenter uses tags to discover available subnets
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: "checkout-prod"
    # Alternative: by subnet ID
    # - id: subnet-0abc123

  # Security Groups: same tag-based discovery logic
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: "checkout-prod"

  # Kubelet: kubelet configuration on nodes (moved from NodePool to EC2NodeClass)
  kubelet:
    maxPods: 110      # increase if using Prefix Delegation (e.g., 737 for m5.xlarge)
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

  # Block device mapping: root volume size and encryption
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 50Gi
        volumeType: gp3
        encrypted: true
        iops: 3000
        throughput: 125

  # Additional tags on all created nodes
  tags:
    Environment: production
    ManagedBy: karpenter

  # userData: additional script executed during bootstrap (RARE — prefer custom AMI)
  # userData: |
  #   #!/bin/bash
  #   echo "custom init" >> /var/log/init.log
```

---

## 4. CDK Python — Karpenter Installation

```python
"""
CDK Stack to install Karpenter on an existing EKS cluster.
Uses Pod Identity (preferred in v1.13) for the controller role.
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
        # 1. IAM Role for NODES created by Karpenter
        #    (not to be confused with the Karpenter controller role)
        # ──────────────────────────────────────────────────────────────
        node_role = iam.Role(self, "KarpenterNodeRole",
            role_name=f"KarpenterNodeRole-{CLUSTER_NAME}",
            description="IAM role for EC2 nodes created by Karpenter",
            assumed_by=iam.ServicePrincipal("ec2.amazonaws.com"),
            managed_policies=[
                iam.ManagedPolicy.from_aws_managed_policy_name("AmazonEKSWorkerNodePolicy"),
                iam.ManagedPolicy.from_aws_managed_policy_name("AmazonEKS_CNI_Policy"),
                iam.ManagedPolicy.from_aws_managed_policy_name("AmazonEC2ContainerRegistryReadOnly"),
                iam.ManagedPolicy.from_aws_managed_policy_name("AmazonSSMManagedInstanceCore"),
            ],
        )

        # Instance profile (required for EC2 to use the role)
        node_instance_profile = iam.CfnInstanceProfile(self, "KarpenterNodeInstanceProfile",
            instance_profile_name=f"KarpenterNodeRole-{CLUSTER_NAME}",
            roles=[node_role.role_name],
        )

        # Add the node role to aws-auth (EKS Access Entries)
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
        # 2. SQS Queue for interruption handling (Spot + maintenance)
        # ──────────────────────────────────────────────────────────────
        interruption_queue = sqs.Queue(self, "KarpenterInterruptionQueue",
            queue_name=CLUSTER_NAME,    # name must be = cluster name
            retention_period=None,
        )

        # Allow EC2 and SQS to publish interruption events to the queue
        interruption_queue.add_to_resource_policy(iam.PolicyStatement(
            principals=[
                iam.ServicePrincipal("sqs.amazonaws.com"),
                iam.ServicePrincipal("events.amazonaws.com"),
            ],
            actions=["sqs:SendMessage"],
            resources=[interruption_queue.queue_arn],
        ))

        # ──────────────────────────────────────────────────────────────
        # 3. IAM Role for Karpenter Controller (via Pod Identity)
        # ──────────────────────────────────────────────────────────────
        controller_role = iam.Role(self, "KarpenterControllerRole",
            role_name=f"{CLUSTER_NAME}-karpenter",
            description="Karpenter controller role — calls EC2 API to create/terminate nodes",
            assumed_by=iam.ServicePrincipal("pods.eks.amazonaws.com"),
        )
        controller_role.assume_role_policy.add_statements(
            iam.PolicyStatement(
                effect=iam.Effect.ALLOW,
                principals=[iam.ServicePrincipal("pods.eks.amazonaws.com")],
                actions=["sts:AssumeRole", "sts:TagSession"],
            )
        )

        # Node lifecycle policy (RunInstances, TerminateInstances, etc.)
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

        # Resource discovery policy
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

        # Role passthrough for instances (IAMIntegration)
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

        # Interruption queue access
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
        # 4. Pod Identity Association for the controller
        # ──────────────────────────────────────────────────────────────
        eks.CfnPodIdentityAssociation(self, "KarpenterPodIdentity",
            cluster_name=CLUSTER_NAME,
            namespace="kube-system",
            service_account="karpenter",
            role_arn=controller_role.role_arn,
        )

        # ──────────────────────────────────────────────────────────────
        # 5. Tags on subnets and security groups for discovery
        #    (Karpenter uses tags to discover resources via EC2 API)
        # ──────────────────────────────────────────────────────────────
        # NOTE: in CDK, add tags via vpc.select_subnets() + tags
        # In practice, easier via eksctl or CLI:
        # aws ec2 create-tags --resources <subnet-ids> \
        #   --tags Key=karpenter.sh/discovery,Value=<cluster-name>

        # ──────────────────────────────────────────────────────────────
        # 6. Karpenter Helm chart
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
                # dnsPolicy: Default if CoreDNS runs on Karpenter nodes
                # "dnsPolicy": "ClusterFirst",  # default
            },
            wait=True,
        )

        CfnOutput(self, "KarpenterControllerRoleArn",
            value=controller_role.role_arn)
        CfnOutput(self, "KarpenterInterruptionQueueUrl",
            value=interruption_queue.queue_url)
```

---

## 5. Python — NodePool Generation by Workload Profile

```python
"""
Generates NodePool + EC2NodeClass manifests for different profiles.
Useful to apply via kubectl apply or via cluster.add_manifest() in CDK.
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
    """Shared EC2NodeClass for multiple NodePools."""
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
    """Generic NodePool with parameterized configurations."""
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
    Generates 3 NodePools + 1 EC2NodeClass for a typical production cluster:
    1. general-od  — on-demand, for critical workloads
    2. general-spot — spot, for interruption-tolerant workloads
    3. gpu         — on-demand p3/g4, for ML workloads (with taint)
    """
    manifests = []

    # Shared EC2NodeClass
    manifests.append(render_ec2_node_class("default", cluster_name))
    # EC2NodeClass for GPU (larger volume)
    manifests.append(render_ec2_node_class("gpu", cluster_name, root_volume_size_gi=100))

    # NodePool 1: On-demand for critical workloads
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

    # NodePool 2: Spot for tolerant workloads (batch, workers)
    manifests.append(render_nodepool(
        name="general-spot",
        node_class_name="default",
        capacity_types=["spot"],
        instance_categories=["c", "m", "r"],
        min_instance_categories=3,   # greater diversity = lower interruption risk
        expire_after="168h",         # 7 days (shorter for spot)
        consolidation_policy="WhenEmptyOrUnderutilized",
        consolidate_after="30s",     # consolidate faster for Spot
        cpu_limit="1000",
        labels={"karpenter.sh/capacity-type-preference": "spot"},
        weight=5,                    # lower weight = second option
    ))

    # NodePool 3: GPU with taint to isolate ML workloads
    manifests.append({
        "apiVersion": "karpenter.sh/v1",
        "kind": "NodePool",
        "metadata": {"name": "gpu"},
        "spec": {
            "template": {
                "spec": {
                    "nodeClassRef": {"group": "karpenter.k8s.aws", "kind": "EC2NodeClass", "name": "gpu"},
                    "taints": [{"key": "nvidia.com/gpu", "value": "true", "effect": "NoSchedule"}],
                    "expireAfter": "Never",   # GPU nodes don't expire (expensive to replace)
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
            "weight": 20,   # higher weight = priority for GPU pods
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

## 6. CLI — Karpenter Installation and Operation

```bash
# ═══════════════════════════════════════════════════════════════
# Initial setup
# ═══════════════════════════════════════════════════════════════

export CLUSTER_NAME="checkout-prod"
export KARPENTER_VERSION="1.13.0"
export K8S_VERSION="1.36"
export AWS_REGION="us-east-1"
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export AWS_PARTITION="aws"
export KARPENTER_NAMESPACE="kube-system"

# Get the optimized EKS AMI version (for EC2NodeClass alias)
export ALIAS_VERSION=$(aws ssm get-parameter \
  --name "/aws/service/eks/optimized-ami/${K8S_VERSION}/amazon-linux-2023/x86_64/standard/recommended/image_id" \
  --query Parameter.Value \
  | xargs aws ec2 describe-images --query 'Images[0].Name' --image-ids \
  | sed -r 's/^.*(v[[:digit:]]+).*$/\1/')

echo "AMI alias version: al2023@${ALIAS_VERSION}"

# Create service-linked role for Spot (required if never used in the account)
aws iam create-service-linked-role --aws-service-name spot.amazonaws.com || true

# ─────────────────────────────────────────────────────────────
# Tag subnets and security groups for Karpenter discovery
# ─────────────────────────────────────────────────────────────

# Get the cluster's private subnets
SUBNET_IDS=$(aws eks describe-cluster \
  --name "$CLUSTER_NAME" \
  --query 'cluster.resourcesVpcConfig.subnetIds[]' \
  --output text)

# Tag subnets for discovery
aws ec2 create-tags \
  --resources $SUBNET_IDS \
  --tags Key=karpenter.sh/discovery,Value="$CLUSTER_NAME"

# Tag the cluster security group
CLUSTER_SG=$(aws eks describe-cluster \
  --name "$CLUSTER_NAME" \
  --query 'cluster.resourcesVpcConfig.clusterSecurityGroupId' \
  --output text)

aws ec2 create-tags \
  --resources "$CLUSTER_SG" \
  --tags Key=karpenter.sh/discovery,Value="$CLUSTER_NAME"

# ─────────────────────────────────────────────────────────────
# Install Karpenter via Helm (OCI registry)
# ─────────────────────────────────────────────────────────────

# Logout first for anonymous pull from public ECR
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

# Verify installation
kubectl get pods -n kube-system -l app.kubernetes.io/name=karpenter
kubectl get crd | grep karpenter

# ─────────────────────────────────────────────────────────────
# Create EC2NodeClass and NodePool
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

# Verify NodePool status
kubectl get nodepool
kubectl describe nodepool default
# Check: status.conditions.type=Ready should be True

# ═══════════════════════════════════════════════════════════════
# Test scale-up and scale-down
# ═══════════════════════════════════════════════════════════════

# Test deployment (pause containers, 1 CPU each)
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

# Scale up: 5 pods = 5 vCPUs → Karpenter provisions a node
kubectl scale deployment inflate --replicas 5

# Watch Karpenter logs in real time
kubectl logs -f -n kube-system \
  -l app.kubernetes.io/name=karpenter \
  -c controller \
  --since=1m | grep -E "provisioned|launched|registered|scheduled"

# Verify created NodeClaim
kubectl get nodeclaims
kubectl describe nodeclaim <name>   # shows which instance type was chosen

# Verify new node
kubectl get nodes -L karpenter.sh/nodepool,node.kubernetes.io/instance-type

# Scale down: Karpenter consolidates and terminates the node
kubectl scale deployment inflate --replicas 0

# After ~1 min, the node should be terminated
kubectl get nodes --watch

# ─────────────────────────────────────────────────────────────
# Maintenance operations
# ─────────────────────────────────────────────────────────────

# List nodes by NodePool
kubectl get nodes -l karpenter.sh/nodepool=default

# Resource status consumed by the NodePool
kubectl get nodepool default -o jsonpath='{.status.resources}' | python3 -m json.tool

# Force immediate consolidation of a NodePool (drift)
kubectl annotate nodepool default \
  karpenter.sh/disruption-reason="manual-consolidation" \
  --overwrite

# Protect a specific pod from disruption
kubectl annotate pod <pod-name> karpenter.sh/do-not-disrupt="true"

# Delete a Karpenter node gracefully (drain + terminate EC2)
kubectl delete node <node-name>
# Karpenter has a finalizer that ensures drain before terminating the instance

# View Karpenter metrics (if Prometheus is available)
kubectl port-forward -n kube-system \
  deployment/karpenter 8080:8080

# Access at http://localhost:8080/metrics
# Relevant metrics:
# karpenter_provisioner_scheduling_duration_seconds
# karpenter_nodes_total_daemon_requests
# karpenter_nodes_total_pod_requests
# karpenter_disruption_consolidation_timeouts_total
```

---

## 7. Pitfalls

[FACT] **Karpenter must not manage the nodes where it runs**: if the cluster's only managed node group is removed and the Karpenter controller tries to consolidate the node where it itself runs, there will be a deadlock. Maintain at least 2 nodes in the system managed node group (or use Fargate for `kube-system`).

[FACT] **Overlapping NodePools without defined `weight` cause random behavior**: if two NodePools can schedule the same pod and don't have different weights, Karpenter chooses randomly. Use `spec.weight` or `taints`/`requirements` to make them mutually exclusive.

[FACT] **Spot without instance diversity causes `InsufficientInstanceCapacity`**: pinning only 1 or 2 Spot instance types is risky. Karpenter uses `price-capacity-optimized` — leave at least 10 eligible instance types. The `minValues` on `instance-family` enforces minimum diversity.

[FACT] **Pods without defined `requests` cause incorrect bin-packing**: Karpenter sizes nodes based on pod `requests`, not `limits`. Pods without `requests` are treated as if they consume no resources, leading to undersized nodes and OOM kills. Use `LimitRange` to define defaults per namespace.

[FACT] **`expireAfter: Never` on general workload NodePools accumulates CVEs**: long-lived nodes accumulate vulnerabilities. The default of 720h (30 days) ensures periodic rotation with OS patches.

[FACT] **Karpenter and Node Termination Handler (NTH) must not coexist**: NTH and Karpenter's interruption handling mechanism can conflict when handling Spot events. If Karpenter is configured with `interruptionQueue`, uninstall NTH.

[FACT] **Controller DNS policy**: Karpenter uses `ClusterFirst` by default, which creates a circular dependency if CoreDNS runs on Karpenter-managed nodes. Solution: keep CoreDNS on a fixed node group, OR use `dnsPolicy: Default` on Karpenter.

---

## Reflection Exercise

An EKS cluster with 3 teams has the following workloads:

- **API Team**: critical service `checkout-api`, 20 replicas with `requests: cpu=500m, memory=512Mi`. Cannot be interrupted during business hours (Mon-Fri 8am–8pm). Must run on `c` or `m` generation 5+ instances.
- **ML Team**: training jobs that run at night, tolerate interruption (checkpoints every 5 min), need `p3.2xlarge` or `g4dn.xlarge` GPUs, and should minimize cost.
- **Platform Team**: DaemonSets and observability tools that must run on **all** nodes.

**Answer:**

1. How many NodePools would you create and what is the justification for each? Describe the `requirements`, `taints`, `weight`, and `disruption` of each one.

2. The ML Team wants to use Spot for their GPU jobs. What specific risks exist with Spot for GPUs and how would Karpenter mitigate those risks with the SQS interruption queue?

3. For `checkout-api`, how would you ensure that Karpenter does **not consolidate** nodes during business hours? (Describe using the correct field from the NodePool spec).

4. The platform team wants their observability tools (DaemonSet) to run on **all** nodes created by Karpenter, including GPU nodes. Why do DaemonSets **not need** tolerations for `NoSchedule` taints created by Karpenter? (Hint: `startupTaints` vs `taints`).

5. An architect proposes migrating from Karpenter to EKS Auto Mode. What are the fundamental differences between EKS Auto Mode and Karpenter for this scenario with multiple workload profiles?

---

## References

- [FACT] [Getting Started with Karpenter (v1.13)](https://karpenter.sh/docs/getting-started/getting-started-with-karpenter/) — karpenter.sh
- [FACT] [NodePools — Karpenter v1.13](https://karpenter.sh/docs/concepts/nodepools/) — karpenter.sh
- [FACT] [NodeClasses — Karpenter v1.13](https://karpenter.sh/docs/concepts/nodeclasses/) — karpenter.sh
- [FACT] [Disruption — Karpenter v1.13](https://karpenter.sh/docs/concepts/disruption/) — karpenter.sh
- [FACT] [Karpenter Best Practices — EKS Best Practices](https://aws.github.io/aws-eks-best-practices/karpenter/) — aws.github.io
- [FACT] [Scale cluster compute with Karpenter and Cluster Autoscaler — Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/autoscaling.html) — docs.aws.amazon.com
