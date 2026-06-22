# Session 046 — EKS: Cluster Provisioning with eksctl and VPC CNI Networking

**Estimated duration:** 60 min  
**Prerequisite:** session-015 (ECS Fargate networking and IAM)

---

## Session Objectives

- Write a complete eksctl ClusterConfig YAML (not inline flags) and run `eksctl create cluster -f`
- Configure kubeconfig with `aws eks update-kubeconfig` and understand the authentication mechanism
- Understand the VPC CNI IP allocation model in Secondary IP Mode (ENIs, slots, warm pool)
- Calculate pod density per instance type using the official formula
- Understand Prefix Delegation Mode as an alternative to Secondary IP mode
- Verify cluster with `kubectl get nodes`, `kubectl get pods -A`

---

## 1. EKS Cluster Architecture

[FACT] An EKS cluster consists of:

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

[FACT] Kubernetes versions currently supported by EKS (source: docs.aws.amazon.com/eks, June 2026):

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

[FACT] eksctl uses `apiVersion: eksctl.io/v1alpha5` and `kind: ClusterConfig`. AWS recommends using a config file instead of inline flags in production environments — the YAML can be versioned in Git and the `--dry-run` flag allows inspecting what will be created before applying.

[FACT] eksctl creates **managed node groups** by default (since eksctl 0.31+). For self-managed, use `--managed=false` or specify `nodeGroups:` (instead of `managedNodeGroups:`) in the YAML.

### 2.1 Minimal config — cluster with managed node group

```yaml
# cluster-basic.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: my-cluster
  region: us-east-1
  version: "1.33"         # Kubernetes version — in quotes (string, not number)
  tags:
    project: checkout
    env: prod

# IAM OIDC provider — required for IRSA (IAM Roles for Service Accounts)
iam:
  withOIDC: true

managedNodeGroups:
  - name: ng-workers
    instanceType: m5.xlarge
    desiredCapacity: 3
    minSize: 2
    maxSize: 10
    volumeSize: 100        # GB, gp3 by default
    privateNetworking: true  # nodes in private subnets (recommended)
    labels:
      role: worker
    tags:
      k8s.io/cluster-autoscaler/enabled: "true"
      k8s.io/cluster-autoscaler/my-cluster: "owned"
    iam:
      withAddonPolicies:
        autoScaler: true          # policy for Cluster Autoscaler
        albIngress: true          # policy for AWS Load Balancer Controller
        cloudWatch: true          # policy for Container Insights
```

### 2.2 Complete config — cluster in existing VPC, multiple node groups

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

# Existing VPC (do not create new VPC)
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
  # Private endpoints for kube-apiserver
  clusterEndpoints:
    publicAccess: true    # internet access (needed for external kubectl)
    privateAccess: true   # internal VPC access (needed for nodes to communicate)

iam:
  withOIDC: true

# Managed add-ons
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

# Main node group — application workloads
managedNodeGroups:
  - name: ng-app-workers
    instanceTypes:
      - m5.xlarge
      - m5a.xlarge
      - m5n.xlarge       # multiple types increase Spot capacity availability
    spot: false           # On-Demand for stable workloads
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
      allow: false        # SSM Session Manager instead of direct SSH

  # Node group for infra workloads (monitoring, logging)
  - name: ng-infra
    instanceType: m5.large
    desiredCapacity: 2
    minSize: 2
    maxSize: 5
    privateNetworking: true
    labels:
      role: infra

# Fargate Profile for specific namespaces
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

### 2.3 Essential commands

```bash
# Dry-run: generates complete ClusterConfig with defaults without creating anything
eksctl create cluster -f cluster-production.yaml --dry-run

# Create cluster (takes ~15-20 min)
eksctl create cluster -f cluster-production.yaml

# Verify CloudFormation stacks created by eksctl
aws cloudformation list-stacks \
  --stack-status-filter CREATE_COMPLETE \
  --query 'StackSummaries[?starts_with(StackName, `eksctl-`)].{Name:StackName,Status:StackStatus}' \
  --output table

# Delete cluster (use --wait to wait for full completion)
eksctl delete cluster -f cluster-production.yaml --wait
```

[FACT] eksctl creates the following CloudFormation stacks:
- `eksctl-<name>-cluster` — VPC, subnets (if not existing VPC), control plane
- `eksctl-<name>-nodegroup-<ng-name>` — one stack per node group
- `eksctl-<name>-addon-iamserviceaccount-<namespace>-<sa>` — one per IRSA

---

## 3. kubeconfig — Authentication with aws eks update-kubeconfig

[FACT] EKS uses the authentication plugin `aws eks get-token` (integrated in AWS CLI v2 as exec credential plugin). The token is valid for 15 minutes and is automatically renewed by kubectl when needed.

```bash
# Configure kubeconfig (creates or merges with existing ~/.kube/config)
aws eks update-kubeconfig \
  --region us-east-1 \
  --name checkout-prod

# Use specific IAM role for authentication (useful in CI/CD)
aws eks update-kubeconfig \
  --region us-east-1 \
  --name checkout-prod \
  --role-arn arn:aws:iam::123456789012:role/EKSDeployRole

# Save to separate file (do not merge with ~/.kube/config)
aws eks update-kubeconfig \
  --region us-east-1 \
  --name checkout-prod \
  --kubeconfig ~/.kube/checkout-prod.yaml

# Verify identity being used (before running kubectl)
aws sts get-caller-identity

# Verify kubeconfig setup
kubectl config view --minify

# Test connectivity with the cluster
kubectl get svc
```

[FACT] The `~/.kube/config` file generated by `update-kubeconfig` contains an exec credentials block like this:

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

[FACT] Only the IAM principal that created the cluster has initial access to Kubernetes RBAC (as cluster-admin). To add other principals, you need to create an `EKS Access Entry` (modern API) or edit the `aws-auth` ConfigMap (legacy method, deprecated in recent versions).

```bash
# Modern method (EKS API, recommended from EKS 1.28)
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

## 4. VPC CNI — IP Allocation Model

### 4.1 Secondary IP Mode (default)

[FACT] The Amazon VPC CNI (plugin `aws-node`, a DaemonSet) implements pod networking. In **Secondary IP Mode** (default):

```
┌─────────────────────────────────────────────────────────────┐
│  EC2 Node (e.g.: m5.xlarge)                                 │
│                                                             │
│  ENI 0 (primary ENI)                                        │
│  ├── 10.0.1.10  ← node's primary IP                        │
│  ├── 10.0.1.11  ← secondary IP → Pod A                     │
│  ├── 10.0.1.12  ← secondary IP → Pod B                     │
│  └── ...up to max IPs per ENI                               │
│                                                             │
│  ENI 1 (secondary ENI — warm pool)                          │
│  ├── 10.0.1.25  ← secondary IP → Pod C                     │
│  ├── 10.0.1.26  ← secondary IP (warm — awaiting pod)       │
│  └── ...                                                    │
│                                                             │
│  ENI 2 (secondary ENI — warm ENI)                           │
│  ├── 10.0.1.40  ← secondary IP (warm)                      │
│  └── ...                                                    │
│                                                             │
│  DaemonSet aws-node (ipamd):                                │
│  - manages ENIs and IPs                                     │
│  - maintains warm pool                                      │
│  - invoked by kubelet via CNI binary                        │
└─────────────────────────────────────────────────────────────┘
```

[FACT] Each Pod receives **a VPC IP** — there is no NAT between pods and the VPC network. A Pod with IP 10.0.1.11 is directly accessible from any resource in the VPC (EC2, RDS, Lambda) using that IP.

### 4.2 Pod density formula (Secondary IP Mode)

[FACT] Official AWS formula for maximum Pods per node:

```
max_pods = (ENIs × (IPs_per_ENI - 1)) + 2

The +2 represents kube-proxy and VPC CNI which run in hostNetwork
(do not consume secondary IPs)
```

[FACT] Examples of common instance types (source: aws/amazon-vpc-cni-k8s/misc/eni-max-pods.txt):

```
╔═══════════════╦══════╦════════════╦═══════════╦═════════════════╗
║ Instance Type ║ vCPU ║ Max ENIs   ║ IPs/ENI   ║ Max Pods (formula) ║
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

[FACT] The 110 Pods per node limit is the **kubelet default maximum**, not the VPC CNI limit. Large instances (e.g.: m5.4xlarge with 234 slots via formula) are limited to 110 by the kubelet unless `--max-pods` is explicitly configured. With **Prefix Delegation**, it is possible to exceed 110 on large instances.

### 4.3 Warm Pool — control variables

[FACT] The ipamd maintains a warm pool of IPs/ENIs ready for pods not yet scheduled. The control variables (configured in the `aws-node` DaemonSet):

```
╔════════════════════╦═══════════════════════════════════════════════╗
║ Variable           ║ Meaning                                       ║
╠════════════════════╬═══════════════════════════════════════════════╣
║ WARM_ENI_TARGET    ║ Number of "warm" ENIs (no pods) maintained    ║
║ (default: 1)       ║ beyond ENIs in use. Each warm ENI consumes    ║
║                    ║ IPs from the subnet CIDR even without pods.   ║
╠════════════════════╬═══════════════════════════════════════════════╣
║ WARM_IP_TARGET     ║ Number of warm IPs available. Overrides       ║
║                    ║ WARM_ENI_TARGET when configured.              ║
╠════════════════════╬═══════════════════════════════════════════════╣
║ MINIMUM_IP_TARGET  ║ Minimum IPs allocated at any time.            ║
║                    ║ Front-loads ENIs at node launch.              ║
╚════════════════════╩═══════════════════════════════════════════════╝
```

[FACT] "Warm" IPs still consume addresses from the subnet CIDR even without associated pods. In subnets with /24 (254 IPs), a node group of 10 nodes with WARM_ENI_TARGET=1 can consume up to 50-100 IPs from warm pool alone.

### 4.4 Prefix Delegation Mode (alternative)

[FACT] Instead of allocating individual IPs as secondary IPs, the VPC CNI can allocate a **/28 prefix** (16 IPs) per slot in each ENI. This multiplies pod density by ~16×.

```
Secondary IP Mode:
  ENI with 15 IPs/ENI → 14 pods per ENI

Prefix Delegation Mode:
  ENI with 15 prefixes/ENI × 16 IPs each = 240 IPs → many more pods per ENI
```

[FACT] Prefix Delegation requires **Nitro** instances (does not work on Xen instances like t2, older m4). Enabled via environment variable `ENABLE_PREFIX_DELEGATION=true` in the `aws-node` DaemonSet or via managed add-on configuration.

[FACT] The transition from Secondary IP → Prefix Delegation should be done by creating a **new node group** — do not perform rolling update on existing nodes (can cause inconsistency in the IP count reported to the scheduler).

---

## 5. CDK Python — EKS Cluster

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
        # IAM role for the control plane
        # ──────────────────────────────────────────────────────────────
        cluster_role = iam.Role(self, "ClusterRole",
            assumed_by=iam.ServicePrincipal("eks.amazonaws.com"),
            managed_policies=[
                iam.ManagedPolicy.from_aws_managed_policy_name("AmazonEKSClusterPolicy"),
            ],
        )

        # ──────────────────────────────────────────────────────────────
        # IAM role for nodes (Node Instance Role)
        # ──────────────────────────────────────────────────────────────
        node_role = iam.Role(self, "NodeRole",
            assumed_by=iam.ServicePrincipal("ec2.amazonaws.com"),
            managed_policies=[
                iam.ManagedPolicy.from_aws_managed_policy_name("AmazonEKSWorkerNodePolicy"),
                iam.ManagedPolicy.from_aws_managed_policy_name("AmazonEC2ContainerRegistryReadOnly"),
                # AmazonEKS_CNI_Policy: required for VPC CNI
                # NOTE: recommended to move to IRSA for aws-node DaemonSet in production
                iam.ManagedPolicy.from_aws_managed_policy_name("AmazonEKS_CNI_Policy"),
                # For Container Insights
                iam.ManagedPolicy.from_aws_managed_policy_name("CloudWatchAgentServerPolicy"),
            ],
        )

        # ──────────────────────────────────────────────────────────────
        # EKS Cluster (L2 construct)
        # ──────────────────────────────────────────────────────────────
        cluster = eks.Cluster(self, "CheckoutCluster",
            cluster_name="checkout-prod",
            version=eks.KubernetesVersion.V1_33,
            role=cluster_role,
            vpc=vpc,
            vpc_subnets=[ec2.SubnetSelection(subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS)],
            # Endpoint access: public + private (recommended)
            endpoint_access=eks.EndpointAccess.PUBLIC_AND_PRIVATE,
            # OIDC provider automatically enabled (for IRSA)
            # default_capacity=0: do not create default node group
            # (we will create custom managed node group below)
            default_capacity=0,
            kubectl_layer=None,   # uses default layer
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
            # Default AMI type: AL2_x86_64 (Amazon Linux 2)
            ami_type=eks.NodegroupAmiType.AL2_X86_64,
            capacity_type=eks.CapacityType.ON_DEMAND,
            labels={"role": "app-worker", "tier": "application"},
            tags={"project": "checkout", "env": "prod"},
        )

        # ──────────────────────────────────────────────────────────────
        # VPC CNI Managed Add-on with Prefix Delegation
        # ──────────────────────────────────────────────────────────────
        # L2 add-ons use CfnAddon (L1)
        from aws_cdk import aws_eks as eks_l1
        vpc_cni_addon = eks.CfnAddon(self, "VpcCniAddon",
            cluster_name=cluster.cluster_name,
            addon_name="vpc-cni",
            # Always use latest or specific version compatible with K8s version
            resolve_conflicts_on_update="OVERWRITE",
            configuration_values='{"env":{"ENABLE_PREFIX_DELEGATION":"true","WARM_PREFIX_TARGET":"1"}}',
        )
        # The add-on can only be created after the cluster exists
        vpc_cni_addon.node.add_dependency(cluster)

        # ──────────────────────────────────────────────────────────────
        # Outputs
        # ──────────────────────────────────────────────────────────────
        CfnOutput(self, "ClusterName",
            value=cluster.cluster_name,
            description="EKS cluster name",
        )
        CfnOutput(self, "ClusterEndpoint",
            value=cluster.cluster_endpoint,
        )
        CfnOutput(self, "OidcIssuer",
            value=cluster.cluster_open_id_connect_issuer,
            description="OIDC issuer URL for IRSA",
        )
        CfnOutput(self, "UpdateKubeconfigCmd",
            value=f"aws eks update-kubeconfig --region {self.region} --name {cluster.cluster_name}",
            description="Command to configure kubectl",
        )
```

---

## 6. Python — Pod density verification script by instance type

```python
"""
Calculates pod density per instance type using the VPC CNI formula.
Useful for planning capacity before creating node groups.
"""
from dataclasses import dataclass

# EC2 limits per instance type (ENIs and IPs/ENI)
# Source: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html
# (table "Network Interfaces per Instance Type")
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

KUBELET_MAX_PODS_DEFAULT = 110   # kubelet default limit


@dataclass
class PodDensityResult:
    instance_type: str
    max_enis: int
    ips_per_eni: int
    max_pods_secondary_ip: int       # formula: (ENIs × (IPs/ENI - 1)) + 2
    max_pods_prefix_delegation: int  # prefix /28 = 16 IPs per slot
    max_pods_effective: int          # limited by kubelet to 110 (without extra config)
    is_kubelet_limited: bool


def calc_pod_density(instance_type: str, override_max_pods: int | None = None) -> PodDensityResult:
    """Calculates pod density for an instance type."""
    if instance_type not in EC2_NETWORK_LIMITS:
        raise ValueError(f"Instance type '{instance_type}' not in local database. "
                         "Check https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html")

    max_enis, ips_per_eni = EC2_NETWORK_LIMITS[instance_type]

    # Official VPC CNI formula (Secondary IP Mode)
    max_pods_secondary = (max_enis * (ips_per_eni - 1)) + 2

    # Prefix Delegation: /28 = 16 IPs per slot (prefix)
    # Each slot that previously had 1 IP can now have 16
    # slots per ENI = ips_per_eni (same limit of "slots")
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
    """Prints comparative pod density report."""
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
    """Estimates subnet IP consumption for a node group."""
    if instance_type not in EC2_NETWORK_LIMITS:
        raise ValueError(f"Unknown instance type: {instance_type}")

    max_enis, ips_per_eni = EC2_NETWORK_LIMITS[instance_type]

    # Minimum IPs: each node has primary ENI + pod IPs
    # Maximum IPs: each node has max_enis ENIs × ips_per_eni (including warm pool)
    ips_per_node_min = ips_per_eni             # primary ENI with IPs
    ips_per_node_max = max_enis * ips_per_eni  # all ENIs fully allocated (warm pool)

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

    # Subnet planning for m5.xlarge with 10 nodes
    consumption = calc_subnet_ip_consumption("m5.xlarge", node_count=10)
    print(f"\nSubnet IP consumption for {consumption['node_count']} × {consumption['instance_type']}:")
    print(f"  Min IPs (pods at minimum): {consumption['ips_consumed_min']}")
    print(f"  Max IPs (warm pool fully allocated): {consumption['ips_consumed_max']}")
    print(f"  Recommendation: {consumption['recommendation']}")
```

---

## 7. CLI — Cluster Verification and Diagnostics

```bash
# ─────────────────────────────────────────────────────────
# AFTER eksctl/CDK creates the cluster, configure kubectl:
# ─────────────────────────────────────────────────────────

aws eks update-kubeconfig --region us-east-1 --name checkout-prod

# ─────────────────────────────────────────────────────────
# Verify nodes (Ready = node ready to receive pods)
# ─────────────────────────────────────────────────────────

kubectl get nodes -o wide
# NAME                        STATUS  ROLES  AGE  VERSION      INTERNAL-IP  INSTANCE-ID
# ip-10-0-1-10.us-east-1...  Ready  <none>  5m  v1.33.0-eks  10.0.1.10    i-0abc123

# Check node capacity details (allocatable pods)
kubectl describe node ip-10-0-1-10.us-east-1.compute.internal | grep -A5 "Allocatable"
# Allocatable:
#   cpu:     3920m
#   memory:  15168Mi
#   pods:    58           ← node max pods (calculated by VPC CNI)

# ─────────────────────────────────────────────────────────
# Verify system pods (including VPC CNI and kube-proxy)
# ─────────────────────────────────────────────────────────

kubectl get pods -n kube-system -o wide
# NAMESPACE     NAME              READY   STATUS    NODE
# kube-system   aws-node-xxxxx    1/1     Running   ip-10-0-1-10...   ← VPC CNI DaemonSet
# kube-system   kube-proxy-xxxxx  1/1     Running   ip-10-0-1-10...
# kube-system   coredns-xxxxx     1/1     Running   ip-10-0-1-10...

# ─────────────────────────────────────────────────────────
# Verify IP allocation on the node (ENIs and allocated IPs)
# ─────────────────────────────────────────────────────────

# List node's ENIs (via EC2 API — substitute the node IP)
NODE_IP="10.0.1.10"
aws ec2 describe-network-interfaces \
  --filters "Name=private-ip-address,Values=${NODE_IP}" \
  --query 'NetworkInterfaces[0].Attachment.InstanceId' \
  --output text | xargs -I {} aws ec2 describe-instances \
    --instance-ids {} \
    --query 'Reservations[0].Instances[0].NetworkInterfaces[*].{ENI:NetworkInterfaceId,IPs:PrivateIpAddresses[*].PrivateIpAddress}' \
    --output table

# Verify installed VPC CNI add-on
aws eks describe-addon \
  --cluster-name checkout-prod \
  --addon-name vpc-cni \
  --query 'addon.{Version:addonVersion,Status:status,Config:configurationValues}'

# ─────────────────────────────────────────────────────────
# Verify aws-node DaemonSet configuration
# ─────────────────────────────────────────────────────────

kubectl get daemonset aws-node -n kube-system -o jsonpath='{.spec.template.spec.containers[0].env}' | python3 -m json.tool

# Verify warm pool variables
kubectl get daemonset aws-node -n kube-system \
  -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="WARM_ENI_TARGET")].value}'

# ─────────────────────────────────────────────────────────
# Verify Access Entries (IAM → K8s RBAC authentication)
# ─────────────────────────────────────────────────────────

aws eks list-access-entries --cluster-name checkout-prod \
  --query 'accessEntries[*]' --output table

# Verify who has cluster access
aws eks list-associated-access-policies \
  --cluster-name checkout-prod \
  --principal-arn arn:aws:iam::123456789012:role/DevOpsRole

# ─────────────────────────────────────────────────────────
# Troubleshooting: node not showing as Ready
# ─────────────────────────────────────────────────────────

# 1. Verify if node can communicate with the API server
kubectl get events --sort-by='.lastTimestamp' | tail -20

# 2. Check aws-node (VPC CNI) logs
kubectl logs -n kube-system -l k8s-app=aws-node --tail=50

# 3. Check node conditions
kubectl describe node <node-name> | grep -A20 "Conditions:"

# 4. Verify the node role has required policies
aws iam list-attached-role-policies \
  --role-name <node-role-name> \
  --query 'AttachedPolicies[*].PolicyName' \
  --output table
```

---

## 8. Pitfalls

[FACT] **IAM principal that creates the cluster is the only one with initial access**: if the cluster is created by a CI/CD role and you try to access with your personal identity, you will get `Unauthorized`. Solution: add your identity as an `EKS Access Entry` before needing it, or create the cluster with a shared role.

[FACT] **`aws-auth` ConfigMap vs EKS Access Entries**: older EKS versions used the `aws-auth` ConfigMap to map IAM → K8s RBAC. From EKS 1.28+, the Access Entries API is the recommended method. The ConfigMap still works but is in legacy mode — manually editing `aws-auth` can make the cluster inaccessible if there is a YAML error.

[CONSENSUS] **Subnets that are too small are the most common cause of scaling failure**: each node consumes multiple IPs from the subnet CIDR (due to warm pool). Use /22 or larger subnets for node subnets in production. The `InsufficientFreeAddressesInSubnet` error only appears during scaling, not at cluster creation.

[FACT] **eksctl creates a new VPC by default**: if you don't specify `vpc:` in the config, eksctl creates a new VPC with CIDR 192.168.0.0/16. In corporate environments, always specify an existing VPC to avoid CIDR conflicts and duplicate NAT Gateway costs.

[FACT] **Prefix Delegation is not retroactive on existing nodes**: enabling `ENABLE_PREFIX_DELEGATION=true` in the DaemonSet does not change allocation on already running nodes. Only new nodes (rolling update of node group) will adopt the new mode.

[FACT] **kubectl version compatibility**: kubectl can be at most 1 minor version above or below the cluster version. A cluster 1.33 can use kubectl 1.32, 1.33, or 1.34. Using kubectl 1.30 with cluster 1.33 may work but is not officially supported.

---

## Reflection Exercise

You are planning an EKS cluster for an image processing service with the following characteristics:
- 3 microservices: API (stateless, high load variance), Worker (CPU-intensive, stable load), Redis Sidecar (memory-intensive)
- Requirement: each Worker Pod needs 3 vCPUs and 6 GB RAM
- SLA: 0 downtime during node group updates
- The available private subnet is a /24 (254 IPs)
- Budget: reduce costs without sacrificing availability

**Answer:**

1. Which instance type to choose for Workers (considering 3 vCPU/6GB per pod and pod density)? What is the maximum Worker pods per node?
2. How many nodes fit in the /24 subnet without exhausting IPs (considering warm pool with WARM_ENI_TARGET=1)?
3. How to guarantee 0 downtime during node group rolling update? Which Kubernetes feature and which node group configurations are needed?
4. Is VPC CNI with Secondary IP Mode sufficient for this subnet, or is it worth enabling Prefix Delegation? Justify with calculations.
5. Should the API (high load variance) be in the same node group as Workers? Why?

---

## References

- [FACT] [Get started with Amazon EKS – eksctl](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html) — docs.aws.amazon.com
- [FACT] [Creating and managing clusters – Eksctl User Guide](https://docs.aws.amazon.com/eks/latest/eksctl/creating-and-managing-clusters.html) — docs.aws.amazon.com
- [FACT] [Connect kubectl to an EKS cluster by creating a kubeconfig file](https://docs.aws.amazon.com/eks/latest/userguide/create-kubeconfig.html) — docs.aws.amazon.com
- [FACT] [Amazon VPC CNI – Best Practices Guide](https://docs.aws.amazon.com/eks/latest/best-practices/vpc-cni.html) — docs.aws.amazon.com
- [FACT] [Assign more IP addresses to Amazon EKS nodes with prefixes](https://docs.aws.amazon.com/eks/latest/userguide/cni-increase-ip-addresses.html) — docs.aws.amazon.com
- [FACT] [Assign IPs to Pods with the Amazon VPC CNI](https://docs.aws.amazon.com/eks/latest/userguide/managing-vpc-cni.html) — docs.aws.amazon.com
