# Session 048 — EKS: IRSA and Pod Identity, Workload-Level IAM Without Static Credentials

**Estimated duration:** 60 min  
**Prerequisite:** session-047 (EKS node groups, Fargate, upgrades)

---

## Session Objectives

- Understand the OIDC flow that underpins IRSA (ProjectedServiceAccountToken → STS AssumeRoleWithWebIdentity)
- Create an IAM OIDC Provider for the cluster and configure a ServiceAccount with an IAM Role via IRSA
- Verify that a pod uses the correct credentials without access keys in the manifest
- Configure EKS Pod Identity (add-on + association) and compare with IRSA
- Decide when to use Pod Identity vs IRSA

---

## 1. Why Not Use Static Credentials

[FACT] Before IRSA and Pod Identity, the common approaches to give AWS access to pods were:

```
PROBLEMATIC — do not use in production:

1. Access keys as env vars in the manifest/Secret
   kubectl create secret generic aws-creds --from-literal=AWS_ACCESS_KEY_ID=xxx
   → Credentials don't rotate, exposed in ETCD, visible in git

2. Node IAM Role (kiam/kube2iam)
   → All pods on the node have access to the node role credentials
   → Unrestricted IMDS = any pod can assume the node role
   → Violates the principle of least privilege

CORRECT — use in production:

3. IRSA (IAM Roles for Service Accounts) — via OIDC
4. EKS Pod Identity — via EKS Auth service (newer)
```

---

## 2. IRSA — How It Works (OIDC Flow)

### 2.1 Technical Fundamentals

[FACT] IRSA uses the OpenID Connect (OIDC) standard, added to IAM in 2014. The flow:

```
┌─────────────────────────────────────────────────────────────────────┐
│ Credential acquisition flow via IRSA                                │
│                                                                     │
│  K8s API Server                                                     │
│  ┌──────────────┐                                                   │
│  │ Projected    │ 1. Kubelet mounts projected token in the pod     │
│  │ Service      │    as a volume at /var/run/secrets/eks.amazonaws  │
│  │ Account      │    .com/serviceaccount/token                      │
│  │ Token (JWT)  │                                                   │
│  └──────┬───────┘                                                   │
│         │ Token contains:                                           │
│         │   - iss: https://oidc.eks.us-east-1.amazonaws.com/id/XXXX│
│         │   - sub: system:serviceaccount:ns:sa-name                 │
│         │   - aud: sts.amazonaws.com                                │
│         │   - exp: now + 1h                                         │
│         ▼                                                           │
│  AWS SDK (inside the pod)                                           │
│  ┌──────────────┐                                                   │
│  │ Credential   │ 2. SDK reads env vars injected by the mutating    │
│  │ Provider     │    webhook:                                        │
│  │ Chain        │    AWS_ROLE_ARN=arn:aws:iam::123:role/MyRole      │
│  │              │    AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/  │
│  └──────┬───────┘    eks.amazonaws.com/serviceaccount/token         │
│         │                                                           │
│         │ 3. SDK calls STS AssumeRoleWithWebIdentity                │
│         ▼                                                           │
│  AWS STS                                                            │
│  ┌──────────────┐                                                   │
│  │ Assume Role  │ 4. STS validates the JWT against the cluster's   │
│  │ With Web     │    OIDC provider (EKS public endpoint)            │
│  │ Identity     │    Verifies: iss, sub, aud, signature             │
│  └──────┬───────┘                                                   │
│         │ 5. Returns temporary credentials (AccessKey, SecretKey,   │
│         │    SessionToken) — valid for 1h by default                │
│         ▼                                                           │
│  Pod receives credentials → accesses S3, DynamoDB, SSM etc.        │
└─────────────────────────────────────────────────────────────────────┘
```

[FACT] EKS hosts a **public OIDC discovery endpoint** per cluster: `https://oidc.eks.<region>.amazonaws.com/id/<CLUSTER_ID>`. STS uses this endpoint to verify the JWT signature without needing to call the cluster directly.

[FACT] The OIDC **signing keys** rotate every **7 days**. EKS keeps the old public keys until they expire. External OIDC clients that cache keys need to refresh before expiration.

[FACT] The `ProjectedServiceAccountToken` (implemented in K8s 1.12) is the token mounted in the pod. It has a default expiration of **1 hour**. EKS grants an extended expiration period of **90 days** (to avoid disruptions in SDKs that don't renew well). Requests with tokens > 90 days are rejected by the API server.

### 2.2 IRSA Trust Policy

[FACT] The IAM role's trust policy needs the OIDC provider ARN as the Federated principal and two conditions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE:sub": "system:serviceaccount:production:checkout-api-sa",
          "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

[FACT] To allow **any ServiceAccount in a namespace** (without pinning the SA name), replace `StringEquals` with `StringLike` and use `*` in the SA name:

```json
"StringLike": {
  "oidc.eks.us-east-1.amazonaws.com/id/XXXX:sub": "system:serviceaccount:production:*"
}
```

[FACT] The default size of a trust policy is **2048 characters**. With the long OIDC issuer URL (~100 chars) and each condition consuming ~120 chars, a trust policy typically supports **4 ServiceAccounts** per role (and at most ~8 with a quota increase).

### 2.3 IRSA Limits

[FACT] IAM OIDC providers limit: **100 per AWS account** by default. An account with more than 100 EKS clusters using IRSA hits this limit. In that case, use Pod Identity (which doesn't require an OIDC provider per cluster).

---

## 3. Pod Identity — How It Works

### 3.1 Architecture

[FACT] EKS Pod Identity uses the **EKS Auth Service** (managed by AWS) to assume the role, instead of making STS calls from inside the pod. The result is less STS load and credentials cached at the node level.

```
┌─────────────────────────────────────────────────────────────────────┐
│ Pod Identity Flow                                                   │
│                                                                     │
│  Pod                    Node                   AWS                  │
│  ┌──────────┐           ┌──────────────────┐   ┌────────────────┐   │
│  │ AWS SDK  │ 1. GET    │ Pod Identity     │   │ EKS Auth       │   │
│  │          │──────────▶│ Agent (DaemonSet)│──▶│ Service        │   │
│  │          │  http://  │ 169.254.170.23   │   │ (AWS managed)  │   │
│  │          │  :80      │ port 80/2703     │   │                │   │
│  └──────────┘           └────────┬─────────┘   └───────┬────────┘   │
│                                  │ 2. Agent calls EKS  │            │
│                                  │    Auth API with    │            │
│                                  │    pod service acct │            │
│                                  └─────────────────────┘            │
│  3. EKS Auth assumes the role in STS and returns temp credentials   │
│  4. Agent returns the credentials to the pod's SDK                  │
│                                                                     │
│  env vars injected by the webhook:                                  │
│    AWS_CONTAINER_CREDENTIALS_FULL_URI=http://169.254.170.23/...    │
│    AWS_CONTAINER_AUTHORIZATION_TOKEN_FILE=/var/run/...              │
└─────────────────────────────────────────────────────────────────────┘
```

[FACT] The Pod Identity Agent is a **DaemonSet** (`amazon-eks-pod-identity-agent`) installed as a managed add-on. It runs on hostNetwork and occupies ports **80** and **2703** on the link-local address **169.254.170.23** (IPv4) or `fd00:ec2::23` (IPv6).

[FACT] The role's trust policy for Pod Identity uses a **fixed service principal** — it doesn't reference an OIDC provider or specific cluster:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "pods.eks.amazonaws.com"
      },
      "Action": ["sts:AssumeRole", "sts:TagSession"]
    }
  ]
}
```

[FACT] `sts:TagSession` is required because Pod Identity includes **session tags** with cluster attributes: `eks-cluster-name`, `eks-cluster-arn`, `kubernetes-namespace`, `kubernetes-service-account`. This enables **ABAC** (attribute-based access control) — a single role can be reused by multiple SAs with different effective permissions based on S3/DynamoDB resource tags etc.

[FACT] Pod Identity limits: up to **5,000 associations** per cluster.

### 3.2 Pod Identity — Restrictions

[FACT] Pod Identity is **not supported** on:
- Fargate (Linux and Windows) — use IRSA for Fargate pods
- Windows EC2 nodes — use IRSA
- AWS Outposts, EKS Anywhere, self-managed K8s — use IRSA
- Requires K8s 1.24+ / platform eks.4 for K8s 1.28 (earlier versions already support it on all platform versions)

---

## 4. Comparison: IRSA vs Pod Identity

[FACT] Official comparison table (docs.aws.amazon.com/eks/latest/userguide/service-accounts.html):

```
╔══════════════════════════════════╦═══════════════════════════╦═══════════════════════════╗
║ Attribute                        ║ Pod Identity              ║ IRSA                      ║
╠══════════════════════════════════╬═══════════════════════════╬═══════════════════════════╣
║ Configuration                    ║ EKS API (no K8s objects)  ║ Annotation on ServiceAcct ║
║ Requires OIDC provider per clust ║ NO                        ║ YES                       ║
║ Trust policy updated per cluster ║ NO (fixed service         ║ YES (OIDC ARN per cluster)║
║                                  ║  principal, reusable)     ║                           ║
║ OIDC providers limit per account ║ N/A                       ║ 100 per account           ║
║ Trust policy size limit          ║ N/A (not cluster-bound)   ║ ~4–8 SAs per role         ║
║ STS calls from the pod           ║ NO (EKS Auth assumes)     ║ YES (SDK calls STS)       ║
║ Session tags (ABAC)              ║ YES (cluster, ns, sa)     ║ NO                        ║
║ Role reusable multi-cluster      ║ YES (no changes needed)   ║ NO (must update trust)    ║
║ Cross-account access             ║ Via role chaining         ║ Directly                  ║
║ Fargate support                  ║ NO                        ║ YES                       ║
║ Supported environments           ║ EKS only                  ║ EKS, EKSa, ROSA, self-mgd║
╚══════════════════════════════════╩═══════════════════════════╩═══════════════════════════╝
```

[FACT] **AWS Recommendation**: use Pod Identity whenever possible. Use IRSA when: Fargate pods, cross-account without role chaining, non-EKS environments, or gradual migration.

---

## 5. CDK Python — IRSA and Pod Identity

```python
from aws_cdk import (
    Stack, CfnOutput,
    aws_eks as eks,
    aws_iam as iam,
    aws_s3 as s3,
    aws_dynamodb as dynamodb,
)
from constructs import Construct


class IrsaStack(Stack):
    """
    IRSA: IAM Roles for Service Accounts
    Prerequisite: cluster with OIDC enabled (withOIDC: true in eksctl or
                  eks.Cluster with openIdConnectProvider automatically created by the L2)
    """
    def __init__(self, scope: Construct, construct_id: str,
                 cluster: eks.Cluster, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # ──────────────────────────────────────────────────────────────
        # AWS resources that the workload will access
        # ──────────────────────────────────────────────────────────────
        config_bucket = s3.Bucket(self, "ConfigBucket",
            bucket_name=f"checkout-config-{self.account}",
        )

        orders_table = dynamodb.Table(self, "OrdersTable",
            table_name="orders",
            partition_key=dynamodb.Attribute(name="orderId", type=dynamodb.AttributeType.STRING),
        )

        # ──────────────────────────────────────────────────────────────
        # IAM Role with IRSA trust policy
        # The CDK L2 eks.Cluster.add_service_account() automatically creates:
        #   1. IAM Role with trust policy referencing the cluster's OIDC provider
        #   2. Kubernetes ServiceAccount with annotation eks.amazonaws.com/role-arn
        # ──────────────────────────────────────────────────────────────
        checkout_sa = cluster.add_service_account("CheckoutApiSA",
            name="checkout-api-sa",
            namespace="production",
            # CDK creates the trust policy with StringEquals for namespace + SA name
        )

        # Minimum permissions for the workload
        config_bucket.grant_read(checkout_sa)
        orders_table.grant_read_write_data(checkout_sa)

        # SSM Parameter Store — application namespace parameters
        checkout_sa.role.add_to_policy(iam.PolicyStatement(
            actions=["ssm:GetParameter", "ssm:GetParametersByPath"],
            resources=[
                f"arn:aws:ssm:{self.region}:{self.account}:parameter/checkout/prod/*"
            ],
        ))

        # Secrets Manager — database secret
        checkout_sa.role.add_to_policy(iam.PolicyStatement(
            actions=["secretsmanager:GetSecretValue"],
            resources=[
                f"arn:aws:secretsmanager:{self.region}:{self.account}:secret:checkout/prod/db*"
            ],
        ))

        # ──────────────────────────────────────────────────────────────
        # SA for VPC CNI (separate IRSA, not inheriting node role)
        # Best practice: isolate CNI permissions from the node role
        # ──────────────────────────────────────────────────────────────
        vpc_cni_sa = cluster.add_service_account("VpcCniSA",
            name="aws-node",
            namespace="kube-system",
        )
        vpc_cni_sa.role.add_managed_policy(
            iam.ManagedPolicy.from_aws_managed_policy_name("AmazonEKS_CNI_Policy")
        )

        # ──────────────────────────────────────────────────────────────
        # Deployment that references the ServiceAccount with IRSA
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
                        # Reference to the ServiceAccount with IRSA — that's all you need
                        # The webhook automatically injects:
                        #   AWS_ROLE_ARN
                        #   AWS_WEB_IDENTITY_TOKEN_FILE
                        "serviceAccountName": "checkout-api-sa",
                        "containers": [{
                            "name": "api",
                            "image": "my-ecr.dkr.ecr.us-east-1.amazonaws.com/checkout-api:v1",
                            "env": [
                                # Force regional STS endpoint (avoid global, which has higher latency)
                                {"name": "AWS_STS_REGIONAL_ENDPOINTS", "value": "regional"},
                                {"name": "AWS_DEFAULT_REGION", "value": "us-east-1"},
                            ],
                            "resources": {
                                "requests": {"cpu": "500m", "memory": "512Mi"},
                                "limits":   {"cpu": "2", "memory": "2Gi"},
                            },
                        }],
                    },
                },
            },
        })

        CfnOutput(self, "CheckoutSARoleArn",
            value=checkout_sa.role.role_arn,
            description="IAM Role ARN for the checkout-api ServiceAccount",
        )


class PodIdentityStack(Stack):
    """
    Pod Identity: newer approach, no OIDC per cluster.
    Requires the 'eks-pod-identity-agent' add-on installed on the cluster.
    """
    def __init__(self, scope: Construct, construct_id: str,
                 cluster: eks.Cluster, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        config_bucket = s3.Bucket(self, "ConfigBucketPodId",
            bucket_name=f"checkout-config-podid-{self.account}",
        )

        # ──────────────────────────────────────────────────────────────
        # IAM Role with Pod Identity trust policy (fixed service principal)
        # This role can be reused across multiple clusters without changes
        # ──────────────────────────────────────────────────────────────
        pod_identity_role = iam.Role(self, "CheckoutPodIdentityRole",
            role_name="CheckoutApiPodIdentityRole",
            description="Role for checkout-api via EKS Pod Identity (reusable multi-cluster)",
            assumed_by=iam.CompositePrincipal(
                iam.ServicePrincipal("pods.eks.amazonaws.com"),
            ),
        )
        # IMPORTANT: TagSession is required for Pod Identity to work with session tags
        pod_identity_role.assume_role_policy.add_statements(
            iam.PolicyStatement(
                effect=iam.Effect.ALLOW,
                principals=[iam.ServicePrincipal("pods.eks.amazonaws.com")],
                actions=["sts:AssumeRole", "sts:TagSession"],
            )
        )

        config_bucket.grant_read(pod_identity_role)
        pod_identity_role.add_to_policy(iam.PolicyStatement(
            actions=["ssm:GetParameter", "ssm:GetParametersByPath"],
            resources=[f"arn:aws:ssm:{self.region}:{self.account}:parameter/checkout/*"],
        ))

        # ──────────────────────────────────────────────────────────────
        # Pod Identity Association: role ↔ ServiceAccount mapping
        # Done via EKS API (there's no Kubernetes object to create)
        # CDK L1 CfnPodIdentityAssociation
        # ──────────────────────────────────────────────────────────────
        pod_id_association = eks.CfnPodIdentityAssociation(self, "CheckoutPodIdAssoc",
            cluster_name=cluster.cluster_name,
            namespace="production",
            service_account="checkout-api-sa",    # SA that already exists in the cluster
            role_arn=pod_identity_role.role_arn,
        )

        # ──────────────────────────────────────────────────────────────
        # Pod Identity Agent add-on (installs the DaemonSet on the cluster)
        # Must be installed BEFORE the associations
        # ──────────────────────────────────────────────────────────────
        pod_id_agent_addon = eks.CfnAddon(self, "PodIdentityAgentAddon",
            cluster_name=cluster.cluster_name,
            addon_name="eks-pod-identity-agent",
            resolve_conflicts_on_update="OVERWRITE",
        )
        # Ensure ordering: add-on installed before the association
        pod_id_association.node.add_dependency(pod_id_agent_addon)

        # ──────────────────────────────────────────────────────────────
        # Kubernetes ServiceAccount — no IAM role annotation
        # With Pod Identity, the association is done outside K8s
        # ──────────────────────────────────────────────────────────────
        cluster.add_manifest("CheckoutSAPodId", {
            "apiVersion": "v1",
            "kind": "ServiceAccount",
            "metadata": {
                "name": "checkout-api-sa",
                "namespace": "production",
                # NO annotation eks.amazonaws.com/role-arn (difference from IRSA)
            },
        })

        CfnOutput(self, "PodIdentityRoleArn",
            value=pod_identity_role.role_arn,
        )
```

---

## 6. Python — Runtime Credential Verification

```python
"""
Application code that uses credentials injected via IRSA or Pod Identity.
No modification is needed in the code — the AWS SDK uses the default credential chain.
"""
import os
import boto3
import json
import logging

logger = logging.getLogger(__name__)

# SDK automatically uses:
# IRSA: AWS_ROLE_ARN + AWS_WEB_IDENTITY_TOKEN_FILE
# Pod Identity: AWS_CONTAINER_CREDENTIALS_FULL_URI + AWS_CONTAINER_AUTHORIZATION_TOKEN_FILE
# No extra line of code needed for IRSA or Pod Identity


def verify_credentials() -> dict:
    """
    Verifies which identity the pod is using.
    Useful for debugging and asserting in tests.
    """
    sts = boto3.client("sts")
    identity = sts.get_caller_identity()

    # Identifies whether it's using IRSA, Pod Identity, or node role
    role_arn = identity["Arn"]
    account = identity["Account"]

    credential_type = "unknown"
    if os.environ.get("AWS_WEB_IDENTITY_TOKEN_FILE"):
        credential_type = "IRSA"
    elif os.environ.get("AWS_CONTAINER_CREDENTIALS_FULL_URI"):
        credential_type = "Pod Identity"
    elif "assumed-role" not in role_arn:
        credential_type = "Node Role (WARNING: violates least privilege)"

    return {
        "account": account,
        "role_arn": role_arn,
        "credential_type": credential_type,
        "user_id": identity["UserId"],
    }


def access_s3_with_irsa(bucket_name: str, key: str) -> str:
    """Reads an object from S3. Uses automatically injected credentials."""
    s3 = boto3.client("s3")
    response = s3.get_object(Bucket=bucket_name, Key=key)
    content = response["Body"].read().decode("utf-8")
    logger.info("S3 read successful: s3://%s/%s", bucket_name, key)
    return content


def access_dynamodb_with_irsa(table_name: str, item_key: dict) -> dict | None:
    """Reads an item from DynamoDB."""
    dynamodb = boto3.resource("dynamodb")
    table = dynamodb.Table(table_name)
    response = table.get_item(Key=item_key)
    return response.get("Item")


def get_ssm_param(name: str) -> str:
    """Reads an SSM parameter. Credentials via IRSA/Pod Identity."""
    ssm = boto3.client("ssm")
    response = ssm.get_parameter(Name=name, WithDecryption=True)
    return response["Parameter"]["Value"]


def get_secret(secret_arn: str) -> dict:
    """Reads a secret from Secrets Manager."""
    sm = boto3.client("secretsmanager")
    response = sm.get_secret_value(SecretId=secret_arn)
    return json.loads(response["SecretString"])


# Lambda handler / app init
def app_init():
    """Called at application startup to verify configuration."""
    creds = verify_credentials()
    if creds["credential_type"] == "Node Role (WARNING: violates least privilege)":
        logger.error("SECURITY WARNING: pod is using node IAM role — configure IRSA or Pod Identity")

    logger.info(
        "Running as: %s (%s) in account %s",
        creds["role_arn"],
        creds["credential_type"],
        creds["account"],
    )


# ABAC verification with Pod Identity session tags
def check_abac_access_example() -> None:
    """
    Pod Identity injects session tags in the role assumption:
      aws:RequestedRegion, eks-cluster-name, kubernetes-namespace,
      kubernetes-service-account

    Example IAM policy that uses ABAC to restrict bucket access
    by namespace (without needing a role per namespace):

    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::checkout-data-${aws:PrincipalTag/kubernetes-namespace}/*"
    }

    With this, a single role can be used by SAs in different namespaces,
    but each SA only accesses the prefix of its own namespace.
    """
    pass
```

---

## 7. CLI — Complete IRSA and Pod Identity Setup

```bash
# ═══════════════════════════════════════════════════════════════
# IRSA — Step-by-step setup
# ═══════════════════════════════════════════════════════════════

CLUSTER_NAME="checkout-prod"
REGION="us-east-1"
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
NAMESPACE="production"
SA_NAME="checkout-api-sa"
ROLE_NAME="CheckoutApiIRSARole"

# Step 1: Create OIDC provider for the cluster
# (only needed once per cluster)
eksctl utils associate-iam-oidc-provider \
  --cluster "$CLUSTER_NAME" \
  --region "$REGION" \
  --approve

# Verify OIDC provider was created
aws iam list-open-id-connect-providers --query 'OpenIDConnectProviderList[*].Arn'

# Get the cluster's OIDC issuer URL
OIDC_URL=$(aws eks describe-cluster \
  --name "$CLUSTER_NAME" \
  --region "$REGION" \
  --query "cluster.identity.oidc.issuer" \
  --output text | sed 's|https://||')

echo "OIDC Provider: $OIDC_URL"
# Output: oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE

# Step 2: Create trust policy with the OIDC provider
cat > trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/${OIDC_URL}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "${OIDC_URL}:sub": "system:serviceaccount:${NAMESPACE}:${SA_NAME}",
          "${OIDC_URL}:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
EOF

# Step 3: Create IAM Role
aws iam create-role \
  --role-name "$ROLE_NAME" \
  --assume-role-policy-document file://trust-policy.json \
  --description "IRSA role for checkout-api"

# Step 4: Attach required policies
aws iam attach-role-policy \
  --role-name "$ROLE_NAME" \
  --policy-arn "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"

# Step 5: Create namespace and ServiceAccount in K8s
kubectl create namespace "$NAMESPACE" --dry-run=client -o yaml | kubectl apply -f -

# Step 6: Create SA with role ARN annotation
kubectl create serviceaccount "$SA_NAME" \
  --namespace "$NAMESPACE" \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl annotate serviceaccount "$SA_NAME" \
  --namespace "$NAMESPACE" \
  eks.amazonaws.com/role-arn="arn:aws:iam::${ACCOUNT_ID}:role/${ROLE_NAME}"

# Verify annotation
kubectl describe serviceaccount "$SA_NAME" -n "$NAMESPACE"
# Should show: eks.amazonaws.com/role-arn: arn:aws:iam::123:role/CheckoutApiIRSARole

# ─────────────────────────────────────────────────────────────
# Verify that the pod uses the correct credentials
# ─────────────────────────────────────────────────────────────

# Create a test pod that uses the SA
kubectl run test-irsa \
  --image=amazon/aws-cli \
  --namespace="$NAMESPACE" \
  --serviceaccount="$SA_NAME" \
  --restart=Never \
  --command -- aws sts get-caller-identity

# Wait for completion and check output
kubectl logs test-irsa -n "$NAMESPACE"
# Expected: "Arn": "arn:aws:sts::123:assumed-role/CheckoutApiIRSARole/..."

# Verify env vars injected by the webhook
kubectl exec -n "$NAMESPACE" test-irsa -- env | grep AWS
# Expected:
# AWS_ROLE_ARN=arn:aws:iam::123:role/CheckoutApiIRSARole
# AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token

# Verify mounted token
kubectl exec -n "$NAMESPACE" test-irsa -- cat /var/run/secrets/eks.amazonaws.com/serviceaccount/token | \
  cut -d. -f2 | base64 -d 2>/dev/null | python3 -m json.tool
# Shows the JWT payload: iss, sub, aud, exp

kubectl delete pod test-irsa -n "$NAMESPACE"
```

```bash
# ═══════════════════════════════════════════════════════════════
# Pod Identity — Setup
# ═══════════════════════════════════════════════════════════════

POD_ID_ROLE="CheckoutApiPodIdentityRole"

# Step 1: Install the Pod Identity Agent add-on
aws eks create-addon \
  --cluster-name "$CLUSTER_NAME" \
  --addon-name eks-pod-identity-agent \
  --region "$REGION"

# Wait for the add-on to become ACTIVE
aws eks wait addon-active \
  --cluster-name "$CLUSTER_NAME" \
  --addon-name eks-pod-identity-agent \
  --region "$REGION"

# Verify DaemonSet is installed
kubectl get daemonset -n kube-system eks-pod-identity-agent

# Step 2: Create IAM Role with Pod Identity trust policy (fixed service principal)
cat > pod-identity-trust.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "pods.eks.amazonaws.com"
      },
      "Action": ["sts:AssumeRole", "sts:TagSession"]
    }
  ]
}
EOF

aws iam create-role \
  --role-name "$POD_ID_ROLE" \
  --assume-role-policy-document file://pod-identity-trust.json \
  --description "Pod Identity role for checkout-api (reusable multi-cluster)"

aws iam attach-role-policy \
  --role-name "$POD_ID_ROLE" \
  --policy-arn "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"

# Step 3: Create Pod Identity Association via EKS API
# (no K8s changes needed — no annotation)
aws eks create-pod-identity-association \
  --cluster-name "$CLUSTER_NAME" \
  --namespace "$NAMESPACE" \
  --service-account "$SA_NAME" \
  --role-arn "arn:aws:iam::${ACCOUNT_ID}:role/${POD_ID_ROLE}" \
  --region "$REGION"

# List associations
aws eks list-pod-identity-associations \
  --cluster-name "$CLUSTER_NAME" \
  --region "$REGION" \
  --query 'associations[*].{NS:namespace,SA:serviceAccount,Role:associationArn}'

# Verify that the SA has no annotation (Pod Identity doesn't need one)
kubectl describe serviceaccount "$SA_NAME" -n "$NAMESPACE"
# Should NOT have eks.amazonaws.com/role-arn

# Verify credentials (same test as IRSA)
kubectl run test-pod-id \
  --image=amazon/aws-cli \
  --namespace="$NAMESPACE" \
  --serviceaccount="$SA_NAME" \
  --restart=Never \
  --command -- aws sts get-caller-identity

kubectl logs test-pod-id -n "$NAMESPACE"
# Expected: "Arn": "arn:aws:sts::123:assumed-role/CheckoutApiPodIdentityRole/..."

# Verify injected env vars (different from IRSA)
kubectl exec -n "$NAMESPACE" test-pod-id -- env | grep AWS
# Expected:
# AWS_CONTAINER_CREDENTIALS_FULL_URI=http://169.254.170.23/v1/credentials
# AWS_CONTAINER_AUTHORIZATION_TOKEN_FILE=/var/run/secrets/pods.eks.amazonaws.com/serviceaccount/eks-pod-identity-token

kubectl delete pod test-pod-id -n "$NAMESPACE"

# ─────────────────────────────────────────────────────────────
# ABAC with Pod Identity — verify session tags
# ─────────────────────────────────────────────────────────────

# When the pod does AssumeRole via Pod Identity, STS includes session tags.
# Verify the session tags:
kubectl run test-tags \
  --image=amazon/aws-cli \
  --namespace="$NAMESPACE" \
  --serviceaccount="$SA_NAME" \
  --restart=Never \
  --command -- aws sts get-caller-identity --query 'Arn'

# The session ARN includes the encoded tags:
# arn:aws:sts::123:assumed-role/CheckoutApiPodIdentityRole/eks-checkout-prod-...
```

---

## 8. Pitfalls

[FACT] **IRSA requires the pod to use the SA correctly — `serviceAccountName` in the spec**: if the pod uses the `default` ServiceAccount (the default when not specified), it won't receive IRSA credentials even if the annotated SA exists in the namespace.

[FACT] **The mutating webhook must be running to inject the env vars**: the `pod-identity-webhook` is part of the control plane managed by EKS — no separate installation needed. If the env vars `AWS_ROLE_ARN` and `AWS_WEB_IDENTITY_TOKEN_FILE` don't appear in the pod, verify that the SA has the correct annotation.

[FACT] **Unrestricted IMDS allows pods without IRSA/Pod Identity to access the node role**: in clusters without IMDS blocking (IMDSv2 not enforced, hop limit > 1), any pod can access the node role credentials via `http://169.254.169.254/latest/meta-data/iam/security-credentials/`. Solution: configure `--metadata-options HttpPutResponseHopLimit=1` in the node group's launch template.

[FACT] **Pod Identity doesn't work on Fargate** — for Fargate pods, always use IRSA. A common mistake is creating a Pod Identity Association expecting it to work for Fargate pods — the pod starts without valid credentials.

[FACT] **Pod Identity is eventually consistent**: there may be a delay of a few seconds after creating or updating an association via the EKS API. Don't create associations in the critical path of bootstrap; do it in separate initialization routines.

[FACT] **Pods using a proxy need to exclude the Pod Identity Agent endpoint from the proxy**: if `HTTP_PROXY` or `HTTPS_PROXY` are configured, the SDK will try to route the call to `169.254.170.23` through the proxy, failing. Add `169.254.170.23` to `NO_PROXY`.

[CONSENSUS] **Remove `AmazonEKS_CNI_Policy` from the node role after configuring IRSA for the VPC CNI**: by default, the CNI uses the node role (via IMDS) to manage ENIs. With a separate IRSA for the aws-node DaemonSet, the CNI policy can be removed from the node role, reducing the blast radius in case of node compromise.

---

## Reflection Exercise

A startup is migrating a monolith to an EKS platform with 3 initial services and expected growth to 50 services in 2 years, distributed across 3 clusters (dev, staging, prod) in the same AWS account:

1. **Checkout API** (EC2 nodes, prod): needs to read/write DynamoDB, publish to SNS, and read secrets from Secrets Manager
2. **Analytics Processor** (Fargate, prod): needs to read from S3 and write to Kinesis
3. **Admin Dashboard** (EC2 nodes, prod): needs cross-account access to an S3 bucket in another account (audit account)

**Answer:**

1. For each service, choose IRSA or Pod Identity and justify based on technical limitations and context (not "I prefer X").
2. How many IAM OIDC providers will be created in the account with 3 clusters? Is this a problem?
3. The Admin Dashboard needs to access the audit account. With Pod Identity, how would you configure this cross-account access (describe the role chaining)? With IRSA, how would it be different?
4. In 2 years, with 50 services across 3 clusters (150 service accounts), a security team wants to use a single IAM Role for multiple services in the same business domain (e.g., "checkout-domain-role"). Which mechanism (IRSA or Pod Identity) facilitates this and why?
5. How do you ensure that no pod can use node role credentials (IMDS) instead of the service-specific credentials?

---

## References

- [FACT] [IAM roles for service accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) — docs.aws.amazon.com
- [FACT] [Learn how EKS Pod Identity grants pods access to AWS services](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html) — docs.aws.amazon.com
- [FACT] [Grant Kubernetes workloads access to AWS using Kubernetes Service Accounts](https://docs.aws.amazon.com/eks/latest/userguide/service-accounts.html) — docs.aws.amazon.com
- [FACT] [Assign IAM roles to Kubernetes service accounts](https://docs.aws.amazon.com/eks/latest/userguide/associate-service-account-role.html) — docs.aws.amazon.com
- [FACT] [Authenticate to another account with IRSA](https://docs.aws.amazon.com/eks/latest/userguide/cross-account-access.html) — docs.aws.amazon.com
