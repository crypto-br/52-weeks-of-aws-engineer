# Sessão 048 — EKS: IRSA e Pod Identity, Workload-Level IAM sem Credenciais Estáticas

**Duração estimada:** 60 min  
**Pré-requisito:** session-047 (EKS node groups, Fargate, upgrades)

---

## Objetivos da sessão

- Entender o fluxo OIDC que fundamenta o IRSA (ProjectedServiceAccountToken → STS AssumeRoleWithWebIdentity)
- Criar IAM OIDC Provider para o cluster e configurar uma ServiceAccount com IAM Role via IRSA
- Verificar que um pod usa as credenciais corretas sem access keys no manifesto
- Configurar EKS Pod Identity (add-on + associação) e comparar com IRSA
- Decidir quando usar Pod Identity vs IRSA

---

## 1. Por que não usar credenciais estáticas

[FATO] Antes de IRSA e Pod Identity, as abordagens comuns para dar acesso AWS a pods eram:

```
PROBLEMÁTICAS — não usar em produção:

1. Access keys como env vars no manifesto/Secret
   kubectl create secret generic aws-creds --from-literal=AWS_ACCESS_KEY_ID=xxx
   → Credenciais não rotacionam, expostas em ETCD, visíveis no git

2. Node IAM Role (kiam/kube2iam)
   → Todos os pods no node têm acesso às credenciais do node role
   → IMDS não restrito = qualquer pod pode assumir o role do node
   → Viola princípio do least privilege

CORRETAS — usar em produção:

3. IRSA (IAM Roles for Service Accounts) — via OIDC
4. EKS Pod Identity — via EKS Auth service (mais recente)
```

---

## 2. IRSA — Como Funciona (fluxo OIDC)

### 2.1 Fundamentos técnicos

[FATO] O IRSA usa o padrão OpenID Connect (OIDC), adicionado ao IAM em 2014. O fluxo:

```
┌─────────────────────────────────────────────────────────────────────┐
│ Fluxo de obtenção de credenciais via IRSA                           │
│                                                                     │
│  K8s API Server                                                     │
│  ┌──────────────┐                                                   │
│  │ Projected    │ 1. Kubelet monta token projetado no pod          │
│  │ Service      │    como volume em /var/run/secrets/eks.amazonaws  │
│  │ Account      │    .com/serviceaccount/token                      │
│  │ Token (JWT)  │                                                   │
│  └──────┬───────┘                                                   │
│         │ Token contém:                                             │
│         │   - iss: https://oidc.eks.us-east-1.amazonaws.com/id/XXXX│
│         │   - sub: system:serviceaccount:ns:sa-name                 │
│         │   - aud: sts.amazonaws.com                                │
│         │   - exp: agora + 1h                                       │
│         ▼                                                           │
│  AWS SDK (dentro do pod)                                            │
│  ┌──────────────┐                                                   │
│  │ Credential   │ 2. SDK lê env vars injetadas pelo mutating        │
│  │ Provider     │    webhook:                                        │
│  │ Chain        │    AWS_ROLE_ARN=arn:aws:iam::123:role/MyRole      │
│  │              │    AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/  │
│  └──────┬───────┘    eks.amazonaws.com/serviceaccount/token         │
│         │                                                           │
│         │ 3. SDK chama STS AssumeRoleWithWebIdentity                │
│         ▼                                                           │
│  AWS STS                                                            │
│  ┌──────────────┐                                                   │
│  │ Assume Role  │ 4. STS valida o JWT contra o OIDC provider do    │
│  │ With Web     │    cluster (endpoint público do EKS)              │
│  │ Identity     │    Verifica: iss, sub, aud, assinatura            │
│  └──────┬───────┘                                                   │
│         │ 5. Retorna credenciais temporárias (AccessKey, SecretKey, │
│         │    SessionToken) — válidas por 1h por padrão              │
│         ▼                                                           │
│  Pod recebe credenciais → acessa S3, DynamoDB, SSM etc.            │
└─────────────────────────────────────────────────────────────────────┘
```

[FATO] O EKS hospeda um **OIDC discovery endpoint público** por cluster: `https://oidc.eks.<region>.amazonaws.com/id/<CLUSTER_ID>`. O STS usa esse endpoint para verificar a assinatura do JWT sem precisar chamar o cluster diretamente.

[FATO] As **signing keys** do OIDC rotacionam a cada **7 dias**. O EKS mantém as chaves públicas antigas até expirarem. Clientes OIDC externos que cachear as chaves precisam atualizar antes da expiração.

[FATO] O `ProjectedServiceAccountToken` (implementado em K8s 1.12) é o token montado no pod. Tem expiração de **1 hora** por padrão. O EKS concede um período de expiração estendido de **90 dias** (para evitar interrupções em SDKs que não renovam bem). Requests com tokens > 90 dias são rejeitados pelo API server.

### 2.2 Trust Policy IRSA

[FATO] A trust policy do IAM role precisa do ARN do OIDC provider como Federated principal e duas conditions:

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

[FATO] Para permitir **qualquer ServiceAccount em um namespace** (sem fixar o nome da SA), substituir `StringEquals` por `StringLike` e usar `*` no nome da SA:

```json
"StringLike": {
  "oidc.eks.us-east-1.amazonaws.com/id/XXXX:sub": "system:serviceaccount:production:*"
}
```

[FATO] O tamanho padrão de uma trust policy é **2048 caracteres**. Com o OIDC issuer URL longo (~100 chars) e cada condition consumindo ~120 chars, uma trust policy suporta tipicamente **4 ServiceAccounts** por role (e no máximo ~8 com aumento de quota).

### 2.3 Limites IRSA

[FATO] Limite de OIDC providers IAM: **100 por conta AWS** por padrão. Uma conta com mais de 100 clusters EKS usando IRSA atinge esse limite. Nesse caso, usar Pod Identity (que não requer OIDC provider por cluster).

---

## 3. Pod Identity — Como Funciona

### 3.1 Arquitetura

[FATO] O EKS Pod Identity usa o **EKS Auth Service** (gerenciado pela AWS) para assumir o role, em vez de fazer STS calls de dentro do pod. O resultado é menos carga de STS e credenciais cacheadas no nível do nó.

```
┌─────────────────────────────────────────────────────────────────────┐
│ Fluxo Pod Identity                                                  │
│                                                                     │
│  Pod                    Node                   AWS                  │
│  ┌──────────┐           ┌──────────────────┐   ┌────────────────┐   │
│  │ AWS SDK  │ 1. GET    │ Pod Identity     │   │ EKS Auth       │   │
│  │          │──────────▶│ Agent (DaemonSet)│──▶│ Service        │   │
│  │          │  http://  │ 169.254.170.23   │   │ (AWS managed)  │   │
│  │          │  :80      │ port 80/2703     │   │                │   │
│  └──────────┘           └────────┬─────────┘   └───────┬────────┘   │
│                                  │ 2. Agent chama EKS  │            │
│                                  │    Auth API com     │            │
│                                  │    pod service acct │            │
│                                  └─────────────────────┘            │
│  3. EKS Auth assume o role no STS e retorna temp credentials        │
│  4. Agent devolve as credenciais ao SDK do pod                      │
│                                                                     │
│  env vars injetadas pelo webhook:                                   │
│    AWS_CONTAINER_CREDENTIALS_FULL_URI=http://169.254.170.23/...    │
│    AWS_CONTAINER_AUTHORIZATION_TOKEN_FILE=/var/run/...              │
└─────────────────────────────────────────────────────────────────────┘
```

[FATO] O Pod Identity Agent é um **DaemonSet** (`amazon-eks-pod-identity-agent`) instalado como managed add-on. Ele roda em hostNetwork e ocupa as portas **80** e **2703** no endereço link-local **169.254.170.23** (IPv4) ou `fd00:ec2::23` (IPv6).

[FATO] A trust policy do role para Pod Identity usa um **service principal fixo** — não referencia OIDC provider nem cluster específico:

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

[FATO] `sts:TagSession` é necessário porque o Pod Identity inclui **session tags** com atributos do cluster: `eks-cluster-name`, `eks-cluster-arn`, `kubernetes-namespace`, `kubernetes-service-account`. Isso habilita **ABAC** (attribute-based access control) — um único role pode ser reusado por múltiplas SAs com permissões efetivas diferentes baseadas em tags dos recursos S3/DynamoDB etc.

[FATO] Limites de Pod Identity: até **5.000 associações** por cluster.

### 3.2 Pod Identity — Restrições

[FATO] Pod Identity **não é suportado** em:
- Fargate (Linux e Windows) — use IRSA para pods Fargate
- Windows EC2 nodes — use IRSA
- AWS Outposts, EKS Anywhere, K8s self-managed — use IRSA
- Requer K8s 1.24+ / plataforma eks.4 para K8s 1.28 (versões anteriores já suportam em todas platform versions)

---

## 4. Comparação: IRSA vs Pod Identity

[FATO] Tabela comparativa oficial (docs.aws.amazon.com/eks/latest/userguide/service-accounts.html):

```
╔══════════════════════════════════╦═══════════════════════════╦═══════════════════════════╗
║ Atributo                         ║ Pod Identity              ║ IRSA                      ║
╠══════════════════════════════════╬═══════════════════════════╬═══════════════════════════╣
║ Configuração                     ║ EKS API (nenhum obj. K8s) ║ Annotation em ServiceAcct ║
║ Requer OIDC provider por cluster ║ NÃO                       ║ SIM                       ║
║ Trust policy atualizada por clust║ NÃO (service principal    ║ SIM (OIDC ARN por cluster)║
║                                  ║  fixo reutilizável)       ║                           ║
║ Limit OIDC providers conta       ║ N/A                       ║ 100 por conta             ║
║ Trust policy size limit          ║ N/A (sem cluster-bound)   ║ ~4–8 SAs por role         ║
║ STS calls no pod                 ║ NÃO (EKS Auth assume)     ║ SIM (SDK chama STS)       ║
║ Session tags (ABAC)              ║ SIM (cluster, ns, sa)     ║ NÃO                       ║
║ Role reusável multi-cluster      ║ SIM (sem alteração)       ║ NÃO (atualizar trust)     ║
║ Cross-account access             ║ Via role chaining         ║ Diretamente               ║
║ Suporte Fargate                  ║ NÃO                       ║ SIM                       ║
║ Ambientes suportados             ║ EKS apenas                ║ EKS, EKSa, ROSA, self-mgd ║
╚══════════════════════════════════╩═══════════════════════════╩═══════════════════════════╝
```

[FATO] **Recomendação AWS**: usar Pod Identity sempre que possível. Usar IRSA quando: pods Fargate, cross-account sem role chaining, ambientes não-EKS, ou migração gradual.

---

## 5. CDK Python — IRSA e Pod Identity

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
    Pré-requisito: cluster com OIDC habilitado (withOIDC: true no eksctl ou
                   eks.Cluster com openIdConnectProvider automaticamente criado pelo L2)
    """
    def __init__(self, scope: Construct, construct_id: str,
                 cluster: eks.Cluster, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # ──────────────────────────────────────────────────────────────
        # Recursos AWS que o workload vai acessar
        # ──────────────────────────────────────────────────────────────
        config_bucket = s3.Bucket(self, "ConfigBucket",
            bucket_name=f"checkout-config-{self.account}",
        )

        orders_table = dynamodb.Table(self, "OrdersTable",
            table_name="orders",
            partition_key=dynamodb.Attribute(name="orderId", type=dynamodb.AttributeType.STRING),
        )

        # ──────────────────────────────────────────────────────────────
        # IAM Role com trust policy IRSA
        # O CDK L2 eks.Cluster.add_service_account() cria automaticamente:
        #   1. IAM Role com trust policy referenciando o OIDC provider do cluster
        #   2. Kubernetes ServiceAccount com annotation eks.amazonaws.com/role-arn
        # ──────────────────────────────────────────────────────────────
        checkout_sa = cluster.add_service_account("CheckoutApiSA",
            name="checkout-api-sa",
            namespace="production",
            # O CDK cria a trust policy com StringEquals para namespace + SA name
        )

        # Permissões mínimas para o workload
        config_bucket.grant_read(checkout_sa)
        orders_table.grant_read_write_data(checkout_sa)

        # SSM Parameter Store — parametros do namespace da aplicação
        checkout_sa.role.add_to_policy(iam.PolicyStatement(
            actions=["ssm:GetParameter", "ssm:GetParametersByPath"],
            resources=[
                f"arn:aws:ssm:{self.region}:{self.account}:parameter/checkout/prod/*"
            ],
        ))

        # Secrets Manager — secret de banco de dados
        checkout_sa.role.add_to_policy(iam.PolicyStatement(
            actions=["secretsmanager:GetSecretValue"],
            resources=[
                f"arn:aws:secretsmanager:{self.region}:{self.account}:secret:checkout/prod/db*"
            ],
        ))

        # ──────────────────────────────────────────────────────────────
        # SA para VPC CNI (IRSA própria, sem herdar node role)
        # Best practice: isolar permissões do CNI do node role
        # ──────────────────────────────────────────────────────────────
        vpc_cni_sa = cluster.add_service_account("VpcCniSA",
            name="aws-node",
            namespace="kube-system",
        )
        vpc_cni_sa.role.add_managed_policy(
            iam.ManagedPolicy.from_aws_managed_policy_name("AmazonEKS_CNI_Policy")
        )

        # ──────────────────────────────────────────────────────────────
        # Deployment que referencia a ServiceAccount com IRSA
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
                        # Referência à ServiceAccount com IRSA — basta isso
                        # O webhook injeta automaticamente:
                        #   AWS_ROLE_ARN
                        #   AWS_WEB_IDENTITY_TOKEN_FILE
                        "serviceAccountName": "checkout-api-sa",
                        "containers": [{
                            "name": "api",
                            "image": "my-ecr.dkr.ecr.us-east-1.amazonaws.com/checkout-api:v1",
                            "env": [
                                # Forçar endpoint regional STS (evitar global, que tem mais latência)
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
            description="IAM Role ARN da ServiceAccount do checkout-api",
        )


class PodIdentityStack(Stack):
    """
    Pod Identity: abordagem mais recente, sem OIDC por cluster.
    Requer add-on 'eks-pod-identity-agent' instalado no cluster.
    """
    def __init__(self, scope: Construct, construct_id: str,
                 cluster: eks.Cluster, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        config_bucket = s3.Bucket(self, "ConfigBucketPodId",
            bucket_name=f"checkout-config-podid-{self.account}",
        )

        # ──────────────────────────────────────────────────────────────
        # IAM Role com trust policy Pod Identity (service principal fixo)
        # Este role pode ser reutilizado em múltiplos clusters sem alteração
        # ──────────────────────────────────────────────────────────────
        pod_identity_role = iam.Role(self, "CheckoutPodIdentityRole",
            role_name="CheckoutApiPodIdentityRole",
            description="Role para checkout-api via EKS Pod Identity (reutilizável multi-cluster)",
            assumed_by=iam.CompositePrincipal(
                iam.ServicePrincipal("pods.eks.amazonaws.com"),
            ),
        )
        # IMPORTANTE: TagSession é obrigatório para Pod Identity funcionar com session tags
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
        # Associação Pod Identity: mapeamento role ↔ ServiceAccount
        # Feito via EKS API (não há objeto Kubernetes para criar)
        # CDK L1 CfnPodIdentityAssociation
        # ──────────────────────────────────────────────────────────────
        pod_id_association = eks.CfnPodIdentityAssociation(self, "CheckoutPodIdAssoc",
            cluster_name=cluster.cluster_name,
            namespace="production",
            service_account="checkout-api-sa",    # SA que já existe no cluster
            role_arn=pod_identity_role.role_arn,
        )

        # ──────────────────────────────────────────────────────────────
        # Add-on Pod Identity Agent (instala o DaemonSet no cluster)
        # Deve ser instalado ANTES das associações
        # ──────────────────────────────────────────────────────────────
        pod_id_agent_addon = eks.CfnAddon(self, "PodIdentityAgentAddon",
            cluster_name=cluster.cluster_name,
            addon_name="eks-pod-identity-agent",
            resolve_conflicts_on_update="OVERWRITE",
        )
        # Garantir ordem: add-on instalado antes da associação
        pod_id_association.node.add_dependency(pod_id_agent_addon)

        # ──────────────────────────────────────────────────────────────
        # Kubernetes ServiceAccount — sem annotation de IAM role
        # Com Pod Identity, a associação é feita fora do K8s
        # ──────────────────────────────────────────────────────────────
        cluster.add_manifest("CheckoutSAPodId", {
            "apiVersion": "v1",
            "kind": "ServiceAccount",
            "metadata": {
                "name": "checkout-api-sa",
                "namespace": "production",
                # SEM annotation eks.amazonaws.com/role-arn (diferença do IRSA)
            },
        })

        CfnOutput(self, "PodIdentityRoleArn",
            value=pod_identity_role.role_arn,
        )
```

---

## 6. Python — Verificação de credenciais em runtime

```python
"""
Código da aplicação que usa credenciais injetadas via IRSA ou Pod Identity.
Nenhuma modificação é necessária no código — o AWS SDK usa a default credential chain.
"""
import os
import boto3
import json
import logging

logger = logging.getLogger(__name__)

# SDK usa automaticamente:
# IRSA: AWS_ROLE_ARN + AWS_WEB_IDENTITY_TOKEN_FILE
# Pod Identity: AWS_CONTAINER_CREDENTIALS_FULL_URI + AWS_CONTAINER_AUTHORIZATION_TOKEN_FILE
# Nenhuma linha de código extra necessária para IRSA ou Pod Identity


def verify_credentials() -> dict:
    """
    Verifica qual identidade o pod está usando.
    Útil para debugging e asserting em testes.
    """
    sts = boto3.client("sts")
    identity = sts.get_caller_identity()

    # Identifica se está usando IRSA, Pod Identity ou node role
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
    """Lê objeto do S3. Usa credenciais injetadas automaticamente."""
    s3 = boto3.client("s3")
    response = s3.get_object(Bucket=bucket_name, Key=key)
    content = response["Body"].read().decode("utf-8")
    logger.info("S3 read successful: s3://%s/%s", bucket_name, key)
    return content


def access_dynamodb_with_irsa(table_name: str, item_key: dict) -> dict | None:
    """Lê item do DynamoDB."""
    dynamodb = boto3.resource("dynamodb")
    table = dynamodb.Table(table_name)
    response = table.get_item(Key=item_key)
    return response.get("Item")


def get_ssm_param(name: str) -> str:
    """Lê parâmetro SSM. Credenciais via IRSA/Pod Identity."""
    ssm = boto3.client("ssm")
    response = ssm.get_parameter(Name=name, WithDecryption=True)
    return response["Parameter"]["Value"]


def get_secret(secret_arn: str) -> dict:
    """Lê secret do Secrets Manager."""
    sm = boto3.client("secretsmanager")
    response = sm.get_secret_value(SecretId=secret_arn)
    return json.loads(response["SecretString"])


# Lambda handler / app init
def app_init():
    """Chamado no startup da aplicação para verificar configuração."""
    creds = verify_credentials()
    if creds["credential_type"] == "Node Role (WARNING: violates least privilege)":
        logger.error("SECURITY WARNING: pod is using node IAM role — configure IRSA or Pod Identity")

    logger.info(
        "Running as: %s (%s) in account %s",
        creds["role_arn"],
        creds["credential_type"],
        creds["account"],
    )


# Verificação de ABAC com Pod Identity session tags
def check_abac_access_example() -> None:
    """
    Pod Identity injeta session tags no role assumption:
      aws:RequestedRegion, eks-cluster-name, kubernetes-namespace,
      kubernetes-service-account

    Exemplo de policy IAM que usa ABAC para restringir acesso a bucket
    por namespace (sem precisar de um role por namespace):

    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::checkout-data-${aws:PrincipalTag/kubernetes-namespace}/*"
    }

    Com isso, um role único pode ser usado por SAs em namespaces diferentes,
    mas cada SA só acessa o prefixo do seu próprio namespace.
    """
    pass
```

---

## 7. CLI — Setup Completo IRSA e Pod Identity

```bash
# ═══════════════════════════════════════════════════════════════
# IRSA — Setup passo a passo
# ═══════════════════════════════════════════════════════════════

CLUSTER_NAME="checkout-prod"
REGION="us-east-1"
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
NAMESPACE="production"
SA_NAME="checkout-api-sa"
ROLE_NAME="CheckoutApiIRSARole"

# Step 1: Criar OIDC provider para o cluster
# (só precisa fazer uma vez por cluster)
eksctl utils associate-iam-oidc-provider \
  --cluster "$CLUSTER_NAME" \
  --region "$REGION" \
  --approve

# Verificar OIDC provider criado
aws iam list-open-id-connect-providers --query 'OpenIDConnectProviderList[*].Arn'

# Obter OIDC issuer URL do cluster
OIDC_URL=$(aws eks describe-cluster \
  --name "$CLUSTER_NAME" \
  --region "$REGION" \
  --query "cluster.identity.oidc.issuer" \
  --output text | sed 's|https://||')

echo "OIDC Provider: $OIDC_URL"
# Saída: oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE

# Step 2: Criar trust policy com o OIDC provider
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

# Step 3: Criar IAM Role
aws iam create-role \
  --role-name "$ROLE_NAME" \
  --assume-role-policy-document file://trust-policy.json \
  --description "IRSA role para checkout-api"

# Step 4: Anexar policies necessárias
aws iam attach-role-policy \
  --role-name "$ROLE_NAME" \
  --policy-arn "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"

# Step 5: Criar namespace e ServiceAccount no K8s
kubectl create namespace "$NAMESPACE" --dry-run=client -o yaml | kubectl apply -f -

# Step 6: Criar SA com annotation do role ARN
kubectl create serviceaccount "$SA_NAME" \
  --namespace "$NAMESPACE" \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl annotate serviceaccount "$SA_NAME" \
  --namespace "$NAMESPACE" \
  eks.amazonaws.com/role-arn="arn:aws:iam::${ACCOUNT_ID}:role/${ROLE_NAME}"

# Verificar annotation
kubectl describe serviceaccount "$SA_NAME" -n "$NAMESPACE"
# Deve mostrar: eks.amazonaws.com/role-arn: arn:aws:iam::123:role/CheckoutApiIRSARole

# ─────────────────────────────────────────────────────────────
# Verificar que o pod usa as credenciais corretas
# ─────────────────────────────────────────────────────────────

# Criar pod de teste que usa a SA
kubectl run test-irsa \
  --image=amazon/aws-cli \
  --namespace="$NAMESPACE" \
  --serviceaccount="$SA_NAME" \
  --restart=Never \
  --command -- aws sts get-caller-identity

# Aguardar completar e ver output
kubectl logs test-irsa -n "$NAMESPACE"
# Esperado: "Arn": "arn:aws:sts::123:assumed-role/CheckoutApiIRSARole/..."

# Verificar env vars injetadas pelo webhook
kubectl exec -n "$NAMESPACE" test-irsa -- env | grep AWS
# Esperado:
# AWS_ROLE_ARN=arn:aws:iam::123:role/CheckoutApiIRSARole
# AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token

# Verificar token montado
kubectl exec -n "$NAMESPACE" test-irsa -- cat /var/run/secrets/eks.amazonaws.com/serviceaccount/token | \
  cut -d. -f2 | base64 -d 2>/dev/null | python3 -m json.tool
# Mostra o payload do JWT: iss, sub, aud, exp

kubectl delete pod test-irsa -n "$NAMESPACE"

# ═══════════════════════════════════════════════════════════════
# Pod Identity — Setup
# ═══════════════════════════════════════════════════════════════

POD_ID_ROLE="CheckoutApiPodIdentityRole"

# Step 1: Instalar o add-on Pod Identity Agent
aws eks create-addon \
  --cluster-name "$CLUSTER_NAME" \
  --addon-name eks-pod-identity-agent \
  --region "$REGION"

# Aguardar add-on ficar ACTIVE
aws eks wait addon-active \
  --cluster-name "$CLUSTER_NAME" \
  --addon-name eks-pod-identity-agent \
  --region "$REGION"

# Verificar DaemonSet instalado
kubectl get daemonset -n kube-system eks-pod-identity-agent

# Step 2: Criar IAM Role com trust policy Pod Identity (service principal fixo)
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
  --description "Pod Identity role para checkout-api (reutilizável multi-cluster)"

aws iam attach-role-policy \
  --role-name "$POD_ID_ROLE" \
  --policy-arn "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"

# Step 3: Criar Pod Identity Association via EKS API
# (não precisa fazer nada no K8s — sem annotation)
aws eks create-pod-identity-association \
  --cluster-name "$CLUSTER_NAME" \
  --namespace "$NAMESPACE" \
  --service-account "$SA_NAME" \
  --role-arn "arn:aws:iam::${ACCOUNT_ID}:role/${POD_ID_ROLE}" \
  --region "$REGION"

# Listar associações
aws eks list-pod-identity-associations \
  --cluster-name "$CLUSTER_NAME" \
  --region "$REGION" \
  --query 'associations[*].{NS:namespace,SA:serviceAccount,Role:associationArn}'

# Verificar que a SA não tem annotation (Pod Identity não precisa)
kubectl describe serviceaccount "$SA_NAME" -n "$NAMESPACE"
# Não deve ter eks.amazonaws.com/role-arn

# Verificar credenciais (mesmo teste do IRSA)
kubectl run test-pod-id \
  --image=amazon/aws-cli \
  --namespace="$NAMESPACE" \
  --serviceaccount="$SA_NAME" \
  --restart=Never \
  --command -- aws sts get-caller-identity

kubectl logs test-pod-id -n "$NAMESPACE"
# Esperado: "Arn": "arn:aws:sts::123:assumed-role/CheckoutApiPodIdentityRole/..."

# Verificar env vars injetadas (diferentes do IRSA)
kubectl exec -n "$NAMESPACE" test-pod-id -- env | grep AWS
# Esperado:
# AWS_CONTAINER_CREDENTIALS_FULL_URI=http://169.254.170.23/v1/credentials
# AWS_CONTAINER_AUTHORIZATION_TOKEN_FILE=/var/run/secrets/pods.eks.amazonaws.com/serviceaccount/eks-pod-identity-token

kubectl delete pod test-pod-id -n "$NAMESPACE"

# ─────────────────────────────────────────────────────────────
# ABAC com Pod Identity — verificar session tags
# ─────────────────────────────────────────────────────────────

# Quando o pod faz AssumeRole via Pod Identity, o STS inclui session tags.
# Verificar as tags na sessão:
kubectl run test-tags \
  --image=amazon/aws-cli \
  --namespace="$NAMESPACE" \
  --serviceaccount="$SA_NAME" \
  --restart=Never \
  --command -- aws sts get-caller-identity --query 'Arn'

# O ARN da sessão inclui as tags codificadas:
# arn:aws:sts::123:assumed-role/CheckoutApiPodIdentityRole/eks-checkout-prod-...
```

---

## 8. Armadilhas (Pitfalls)

[FATO] **IRSA requer que o pod use a SA corretamente — `serviceAccountName` no spec**: se o pod usa a `default` ServiceAccount (padrão quando não especificado), não recebe as credenciais IRSA mesmo que a SA anotada exista no namespace.

[FATO] **O mutating webhook precisa estar rodando para injetar as env vars**: o webhook `pod-identity-webhook` é parte do control plane gerenciado pelo EKS — não precisa instalar separadamente. Se as env vars `AWS_ROLE_ARN` e `AWS_WEB_IDENTITY_TOKEN_FILE` não aparecem no pod, verificar se a SA tem a annotation correta.

[FATO] **IMDS não restrito permite que pods sem IRSA/Pod Identity acessem o node role**: em clusters sem bloqueio de IMDS (IMDSv2 não obrigatório, hop limit > 1), qualquer pod pode acessar as credenciais do node role via `http://169.254.169.254/latest/meta-data/iam/security-credentials/`. Solução: configurar `--metadata-options HttpPutResponseHopLimit=1` no launch template do node group.

[FATO] **Pod Identity não funciona em Fargate** — para pods Fargate, sempre usar IRSA. Um erro comum é criar um Pod Identity Association esperando que funcione para pods Fargate — o pod inicia sem credenciais válidas.

[FATO] **Pod Identity é eventually consistent**: pode haver delay de alguns segundos após criar ou atualizar uma associação via EKS API. Não criar associações em critical path de bootstrap; fazer em rotinas de inicialização separadas.

[FATO] **Pods usando proxy precisam excluir o endpoint do Pod Identity Agent do proxy**: se `HTTP_PROXY` ou `HTTPS_PROXY` estiverem configurados, o SDK tentará rotear a chamada para `169.254.170.23` pelo proxy, falhando. Adicionar `169.254.170.23` ao `NO_PROXY`.

[CONSENSO] **Remover `AmazonEKS_CNI_Policy` do node role após configurar IRSA para o VPC CNI**: por padrão, o CNI usa o node role (via IMDS) para gerenciar ENIs. Com IRSA separada para o aws-node DaemonSet, a CNI policy pode ser removida do node role, reduzindo o blast radius em caso de comprometimento do node.

---

## Exercício de Reflexão

Uma startup está migrando um monolito para uma plataforma EKS com 3 serviços iniciais e crescimento esperado para 50 serviços em 2 anos, distribuídos em 3 clusters (dev, staging, prod) na mesma conta AWS:

1. **Checkout API** (EC2 nodes, prod): precisa ler/escrever no DynamoDB, publicar no SNS, e ler segredos do Secrets Manager
2. **Analytics Processor** (Fargate, prod): precisa ler do S3 e escrever no Kinesis
3. **Admin Dashboard** (EC2 nodes, prod): precisa de acesso cross-account a um bucket S3 em outra conta (conta de auditoria)

**Responda:**

1. Para cada serviço, escolha IRSA ou Pod Identity e justifique baseado nas limitações técnicas e no contexto (não "eu prefiro X").
2. Quantos IAM OIDC providers serão criados na conta com 3 clusters? Isso é um problema?
3. O Admin Dashboard precisa acessar a conta de auditoria. Com Pod Identity, como configurar esse cross-account access (descreva o role chaining)? Com IRSA, como seria diferente?
4. Em 2 anos, com 50 serviços em 3 clusters (150 service accounts), um time de segurança quer usar uma única IAM Role para múltiplos serviços do mesmo domínio de negócio (ex.: "checkout-domain-role"). Qual mecanismo (IRSA ou Pod Identity) facilita isso e por quê?
5. Como garantir que nenhum pod consiga usar credenciais do node role (IMDS) em vez das credenciais específicas do serviço?

---

## Referências

- [FATO] [IAM roles for service accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) — docs.aws.amazon.com
- [FATO] [Learn how EKS Pod Identity grants pods access to AWS services](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html) — docs.aws.amazon.com
- [FATO] [Grant Kubernetes workloads access to AWS using Kubernetes Service Accounts](https://docs.aws.amazon.com/eks/latest/userguide/service-accounts.html) — docs.aws.amazon.com
- [FATO] [Assign IAM roles to Kubernetes service accounts](https://docs.aws.amazon.com/eks/latest/userguide/associate-service-account-role.html) — docs.aws.amazon.com
- [FATO] [Authenticate to another account with IRSA](https://docs.aws.amazon.com/eks/latest/userguide/cross-account-access.html) — docs.aws.amazon.com
