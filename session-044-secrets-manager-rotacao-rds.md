# Sessão 044 — Secrets Manager: Rotação Automática, Lambda Rotators e Integração com RDS

**Duração estimada:** 60 min  
**Pré-requisito:** nenhum (sessão autossuficiente)

---

## Objetivos da sessão

- Entender o ciclo de vida de versões e staging labels (AWSPENDING → AWSCURRENT → AWSPREVIOUS)
- Implementar os quatro steps da rotation function (createSecret, setSecret, testSecret, finishSecret)
- Distinguir Single User vs Alternating Users e quando cada estratégia é adequada
- Configurar rotação nativa para RDS PostgreSQL com Lambda gerenciado pela AWS
- Construir rotador customizado para serviço externo
- Escrever aplicações resilientes a rotação (cache + retry sem downtime)

---

## 1. Fundamentos do Secrets Manager

### 1.1 Versões e Staging Labels

[FATO] Cada segredo no Secrets Manager pode ter **múltiplas versões simultâneas**. Cada versão recebe um ou mais **staging labels** que identificam seu papel no ciclo de rotação.

```
Estados de staging labels durante rotação:

  ┌──────────────────────────────────────────────────────────────┐
  │ Antes da rotação                                             │
  │   versão A: [AWSCURRENT]                                     │
  │   versão B: [AWSPREVIOUS]  (da rotação anterior)            │
  └──────────────────────────────────────────────────────────────┘
  
  ┌──────────────────────────────────────────────────────────────┐
  │ Durante a rotação (entre createSecret e finishSecret)        │
  │   versão A: [AWSCURRENT]                                     │
  │   versão B: [AWSPREVIOUS]                                    │
  │   versão C: [AWSPENDING]   ← nova senha gerada, DB atualizado│
  └──────────────────────────────────────────────────────────────┘
  
  ┌──────────────────────────────────────────────────────────────┐
  │ Após finishSecret (rotação completa)                         │
  │   versão A: [AWSPREVIOUS]  ← era AWSCURRENT                 │
  │   versão B: sem label      ← removida automaticamente        │
  │   versão C: [AWSCURRENT]   ← nova versão promovida          │
  └──────────────────────────────────────────────────────────────┘
```

[FATO] `AWSPENDING` nunca deve ser removido manualmente antes do `finishSecret`. Se `AWSPENDING` existir sem estar no mesmo versionId que `AWSCURRENT`, qualquer nova tentativa de rotação assumirá que uma rotação anterior está em andamento e retornará erro.

### 1.2 Criptografia

[FATO] Todos os segredos são criptografados em repouso. Por padrão, usa a chave gerenciada pela AWS `aws/secretsmanager` (sem custo adicional de KMS, mas sem controle de key policy). Para controle granular, use uma CMK (Customer Managed Key).

[FATO] Se um segredo usa CMK customizada, a Lambda de rotação precisa de permissão `kms:Decrypt` e `kms:GenerateDataKey` na CMK. Use `kms:EncryptionContext:SecretARN` para limitar o acesso da Lambda apenas ao segredo que ela rotaciona.

### 1.3 Preço

[FATO] $0.40/segredo/mês + $0.05 por 10.000 chamadas de API. Os primeiros 30 dias de um segredo novo são gratuitos.

---

## 2. Os Quatro Steps da Rotation Function

[FATO] Toda rotation function — nativa ou customizada — deve implementar quatro métodos invocados em sequência pelo Secrets Manager:

```
Fluxo sequencial de rotação:

  Secrets Manager
       │
       ├──1──► createSecret  → gera nova senha
       │                     → put_secret_value com label AWSPENDING
       │                     → idempotente: verifica se AWSPENDING já existe
       │
       ├──2──► setSecret     → aplica nova senha no banco/serviço de destino
       │                     → usa credenciais AWSCURRENT para conectar
       │                     → muda senha do usuário para o valor em AWSPENDING
       │
       ├──3──► testSecret    → testa conexão usando AWSPENDING
       │                     → lê algo do banco para confirmar que funciona
       │                     → se falhar → rotação para; AWSPENDING permanece
       │
       └──4──► finishSecret  → update_secret_version_stage:
                               move AWSCURRENT → versão antiga vira AWSPREVIOUS
                               versão AWSPENDING → vira nova AWSCURRENT
                               AWSPENDING é removido atomicamente
```

### 2.1 Idempotência e ClientRequestToken

[FATO] O Secrets Manager passa um `ClientRequestToken` (UUID) como `VersionId` para cada chamada de rotação. O `createSecret` deve verificar se uma versão com esse token já existe antes de gerar nova senha — isso garante idempotência se a Lambda for chamada novamente após uma falha parcial.

---

## 3. Estratégias de Rotação

### 3.1 Single User (recomendada para maioria dos casos)

[FATO] Atualiza credenciais de **um único usuário** em um único segredo. A sequência é: gera nova senha → aplica no banco → atualiza segredo.

```
Linha do tempo — Single User:

  t0  createSecret:  AWSPENDING criado com nova senha
  t1  setSecret:     DB muda senha do usuário
  t2  [janela de risco]: DB já tem nova senha, AWSCURRENT ainda tem a antiga
                         → conexões novas com AWSCURRENT falham
  t3  testSecret:    testa AWSPENDING → sucesso
  t4  finishSecret:  AWSPENDING promovido a AWSCURRENT
  t5  [normalizado]: todas as conexões novas usam nova senha

Duração da janela de risco (t1→t4): segundos a milissegundos
Mitigação: exponential backoff + retry na aplicação
```

[FATO] Single User é recomendado para: casos gerais, usuários ad hoc/interativos, e quando o banco não tem suporte para clonar usuários. **RDS Proxy suporta Single User.**

### 3.2 Alternating Users (alta disponibilidade)

[FATO] Mantém dois usuários (ex.: `myuser` e `myuser_clone`) e alterna qual tem a senha atualizada. A rotação funciona assim:

```
Primeira rotação:
  AWSPENDING = {username: "myuser_clone", password: gerada}
  Lambda clona "myuser" → cria "myuser_clone" com mesma permissão
  AWSCURRENT passa a apontar para myuser_clone

Segunda rotação:
  AWSPENDING = {username: "myuser", password: nova_senha}
  Lambda atualiza senha de "myuser" (não precisa clonar)
  AWSCURRENT passa a apontar para myuser

Terceira rotação: idem à primeira (alterna de volta para myuser_clone)
```

[FATO] Alternating Users requer um **segundo segredo** com credenciais de superusuário (que tem permissão de criar/clonar usuários no banco). A Lambda usa o superusuário para clonar e modificar permissões.

[FATO] **Amazon RDS Proxy NÃO suporta Alternating Users** — use Single User quando tiver RDS Proxy.

[FATO] Se as permissões do usuário original forem alteradas após a criação do clone, o usuário clonado **não é atualizado automaticamente** — é necessário atualizar manualmente.

---

## 4. Network Access — Requisito Crítico

[FATO] A Lambda de rotação precisa alcançar **dois endpoints simultaneamente**:

```
Lambda de rotação (na VPC)
    │
    ├──► Secrets Manager endpoint
    │    Opção A: VPC Endpoint (Interface) para secretsmanager — recomendado
    │    Opção B: NAT Gateway + Internet Gateway
    │
    └──► RDS PostgreSQL endpoint (porta 5432)
         Security Group: permitir Lambda SG → RDS SG na porta 5432
```

[FATO] Se a Lambda não conseguir alcançar o endpoint do Secrets Manager, a rotação falha imediatamente com `SecretsManagerServiceException`. A causa mais comum de falha de rotação é problema de rede/VPC.

---

## 5. CDK Python — Rotação Nativa RDS PostgreSQL

```python
from aws_cdk import (
    Stack, Duration, RemovalPolicy,
    aws_ec2 as ec2,
    aws_rds as rds,
    aws_secretsmanager as sm,
    aws_iam as iam,
    aws_lambda as _lambda,
    aws_kms as kms,
)
from constructs import Construct

class SecretsManagerRDSStack(Stack):
    def __init__(self, scope: Construct, construct_id: str, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        vpc = ec2.Vpc(self, "VPC", max_azs=2, nat_gateways=1)

        # KMS CMK para criptografia dos segredos
        secret_key = kms.Key(self, "SecretKey",
            description="CMK para Secrets Manager - RDS credentials",
            enable_key_rotation=True,  # rotação da própria CMK anualmente
        )

        # ──────────────────────────────────────────────────────────────
        # Segredo do superusuário RDS (admin) — necessário para
        # Alternating Users strategy
        # ──────────────────────────────────────────────────────────────
        superuser_secret = sm.Secret(self, "RDSSuperuserSecret",
            secret_name="rds/postgres/superuser",
            description="Credenciais do superusuário RDS PostgreSQL",
            encryption_key=secret_key,
            generate_secret_string=sm.SecretStringGenerator(
                secret_string_template='{"username": "postgres"}',
                generate_string_key="password",
                password_length=32,
                exclude_characters="/@\"\\' ",
                exclude_punctuation=False,
            ),
        )

        # ──────────────────────────────────────────────────────────────
        # Instância RDS PostgreSQL com rotação de credenciais
        # ──────────────────────────────────────────────────────────────
        db_sg = ec2.SecurityGroup(self, "DBSG", vpc=vpc)

        db_instance = rds.DatabaseInstance(self, "PostgresDB",
            engine=rds.DatabaseInstanceEngine.postgres(
                version=rds.PostgresEngineVersion.VER_16
            ),
            instance_type=ec2.InstanceType.of(
                ec2.InstanceClass.T3, ec2.InstanceSize.MEDIUM
            ),
            vpc=vpc,
            vpc_subnets=ec2.SubnetSelection(subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS),
            security_groups=[db_sg],
            credentials=rds.Credentials.from_secret(superuser_secret),
            removal_policy=RemovalPolicy.DESTROY,
            deletion_protection=False,
        )

        # ──────────────────────────────────────────────────────────────
        # Segredo da aplicação (usuário "app_user") — será rotacionado
        # com Alternating Users strategy
        # ──────────────────────────────────────────────────────────────
        app_secret = sm.Secret(self, "AppUserSecret",
            secret_name="rds/postgres/app-user",
            description="Credenciais do usuário da aplicação — rotação automática",
            encryption_key=secret_key,
            generate_secret_string=sm.SecretStringGenerator(
                secret_string_template=(
                    '{"username": "app_user",'
                    f'"host": "{db_instance.db_instance_endpoint_address}",'
                    f'"port": "{db_instance.db_instance_endpoint_port}",'
                    '"dbname": "appdb","engine": "postgres"}'
                ),
                generate_string_key="password",
                password_length=32,
                exclude_characters="/@\"\\' ;",
            ),
        )

        # ──────────────────────────────────────────────────────────────
        # Rotação automática: estratégia Alternating Users
        # HostedRotationLambda = Lambda gerenciada pela AWS (rotação nativa)
        # ──────────────────────────────────────────────────────────────
        app_secret.add_rotation_schedule("RotationSchedule",
            hosted_rotation=sm.HostedRotation.postgre_sql_multi_user(
                function_name="SecretsManagerRDSPostgreSQLRotation",
                master_secret=superuser_secret,  # superusuário para clonar
                vpc=vpc,
                vpc_subnets=ec2.SubnetSelection(
                    subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS
                ),
                security_groups=[ec2.SecurityGroup(self, "RotationLambdaSG", vpc=vpc)],
                exclude_characters="/@\"\\' ;",
            ),
            automatically_after=Duration.days(30),  # rotação a cada 30 dias
        )

        # Permitir Lambda de rotação conectar ao RDS
        db_sg.add_ingress_rule(
            peer=ec2.Peer.ipv4(vpc.vpc_cidr_block),
            connection=ec2.Port.tcp(5432),
            description="Permitir Lambda de rotação acessar PostgreSQL",
        )

        # VPC Endpoint para Secrets Manager — Lambda não precisa de NAT
        vpc.add_interface_endpoint("SecretsManagerEndpoint",
            service=ec2.InterfaceVpcEndpointAwsService.SECRETS_MANAGER,
        )

        # ──────────────────────────────────────────────────────────────
        # Exemplo de rotação Single User (mais simples)
        # Para quando Alternating Users não é necessário
        # ──────────────────────────────────────────────────────────────
        simple_secret = sm.Secret(self, "SimpleSecret",
            secret_name="rds/postgres/simple-user",
            encryption_key=secret_key,
            generate_secret_string=sm.SecretStringGenerator(
                secret_string_template=(
                    '{"username": "simple_user",'
                    f'"host": "{db_instance.db_instance_endpoint_address}"}'
                ),
                generate_string_key="password",
                password_length=32,
                exclude_characters="/@\"\\' ",
            ),
        )

        simple_secret.add_rotation_schedule("SimpleRotationSchedule",
            hosted_rotation=sm.HostedRotation.postgre_sql_single_user(
                function_name="SecretsManagerRDSPostgreSQLSingleUserRotation",
                vpc=vpc,
                vpc_subnets=ec2.SubnetSelection(
                    subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS
                ),
            ),
            automatically_after=Duration.days(30),
        )
```

---

## 6. Python — Rotation Function Customizada (Serviço Externo)

```python
"""
Template de rotation function customizada para serviço externo
(ex.: API key de terceiro, token OAuth, senha de sistema legado).

Deployed via CDK Lambda Function apontado como rotation function no segredo.
"""
import json
import logging
import boto3
from botocore.exceptions import ClientError

logger = logging.getLogger(__name__)
logger.setLevel(logging.INFO)

secretsmanager = boto3.client("secretsmanager")


def handler(event: dict, context) -> None:
    """
    Entry point: Secrets Manager invoca esta função com um evento
    contendo arn, token e step.
    """
    arn   = event["SecretId"]
    token = event["ClientRequestToken"]
    step  = event["Step"]

    # Verificar que o segredo tem a versão com este token
    metadata = secretsmanager.describe_secret(SecretId=arn)
    versions = metadata.get("VersionIdsToStages", {})

    if token not in versions:
        raise ValueError(f"Secret version {token} has no stage for secret {arn}")

    if "AWSCURRENT" in versions[token]:
        # Rotação já foi concluída para este token — idempotente
        logger.info("Version %s already set as AWSCURRENT, nothing to do.", token)
        return

    if "AWSPENDING" not in versions[token]:
        raise ValueError(f"Secret version {token} not set as AWSPENDING for {arn}")

    dispatch = {
        "createSecret": create_secret,
        "setSecret":    set_secret,
        "testSecret":   test_secret,
        "finishSecret": finish_secret,
    }
    if step not in dispatch:
        raise ValueError(f"Invalid step parameter: {step}")

    dispatch[step](secretsmanager, arn, token)


def create_secret(client, arn: str, token: str) -> None:
    """
    Step 1: Gera nova credencial e armazena como AWSPENDING.
    Idempotente: se AWSPENDING já existe com este token, não faz nada.
    """
    # Tentar buscar a versão AWSPENDING existente (para idempotência)
    try:
        client.get_secret_value(SecretId=arn, VersionId=token, VersionStage="AWSPENDING")
        logger.info("createSecret: AWSPENDING version %s already exists. Idempotent skip.", token)
        return
    except client.exceptions.ResourceNotFoundException:
        pass  # AWSPENDING não existe ainda — prosseguir

    # Buscar o segredo atual para obter estrutura
    current = json.loads(
        client.get_secret_value(SecretId=arn, VersionStage="AWSCURRENT")["SecretString"]
    )

    # Gerar nova credencial (ex.: nova API key)
    new_credential = _generate_new_credential(current)

    # Armazenar como AWSPENDING
    client.put_secret_value(
        SecretId=arn,
        ClientRequestToken=token,
        SecretString=json.dumps(new_credential),
        VersionStages=["AWSPENDING"],
    )
    logger.info("createSecret: new version stored as AWSPENDING for %s.", arn)


def set_secret(client, arn: str, token: str) -> None:
    """
    Step 2: Aplica a nova credencial no serviço de destino.
    IMPORTANTE: verificar que AWSCURRENT e AWSPENDING apontam para o mesmo recurso
    antes de modificar (defesa contra confused deputy).
    """
    current_secret = json.loads(
        client.get_secret_value(SecretId=arn, VersionStage="AWSCURRENT")["SecretString"]
    )
    pending_secret = json.loads(
        client.get_secret_value(SecretId=arn, VersionId=token, VersionStage="AWSPENDING")["SecretString"]
    )

    # Validação de segurança: mesmo recurso de destino
    if current_secret.get("endpoint") != pending_secret.get("endpoint"):
        raise ValueError("AWSCURRENT and AWSPENDING point to different endpoints. Aborting.")

    # Aplicar a nova credencial no serviço externo
    _apply_credential_to_service(
        endpoint=pending_secret["endpoint"],
        old_api_key=current_secret["api_key"],
        new_api_key=pending_secret["api_key"],
    )
    logger.info("setSecret: credential updated in service for %s.", arn)


def test_secret(client, arn: str, token: str) -> None:
    """
    Step 3: Valida que a nova credencial (AWSPENDING) funciona.
    """
    pending_secret = json.loads(
        client.get_secret_value(SecretId=arn, VersionId=token, VersionStage="AWSPENDING")["SecretString"]
    )

    # Testar a credencial contra o serviço
    if not _test_credential(pending_secret):
        raise RuntimeError(
            f"testSecret: new credential failed validation for {arn}. "
            "Rotation will not complete."
        )
    logger.info("testSecret: new credential validated successfully for %s.", arn)


def finish_secret(client, arn: str, token: str) -> None:
    """
    Step 4: Promove AWSPENDING a AWSCURRENT atomicamente.
    Secrets Manager automaticamente move a versão anterior para AWSPREVIOUS.
    NÃO remover AWSPENDING manualmente antes desta chamada.
    """
    # Encontrar o versionId atual de AWSCURRENT
    metadata = client.describe_secret(SecretId=arn)
    current_version = next(
        (vid for vid, stages in metadata["VersionIdsToStages"].items()
         if "AWSCURRENT" in stages and vid != token),
        None,
    )

    # Mover AWSCURRENT da versão antiga para a nova versão (token)
    # Esta chamada também remove AWSPENDING atomicamente
    client.update_secret_version_stage(
        SecretId=arn,
        VersionStage="AWSCURRENT",
        MoveToVersionId=token,
        RemoveFromVersionId=current_version,
    )
    logger.info(
        "finishSecret: rotation complete. New version %s is now AWSCURRENT for %s.",
        token, arn
    )


# ── Funções auxiliares do serviço externo ──────────────────────────────────

def _generate_new_credential(current: dict) -> dict:
    """Gera nova credencial mantendo a estrutura do segredo."""
    import secrets
    return {
        **current,
        "api_key": secrets.token_urlsafe(32),
        "generated_at": __import__("datetime").datetime.utcnow().isoformat() + "Z",
    }


def _apply_credential_to_service(endpoint: str, old_api_key: str, new_api_key: str):
    """
    Aplica a nova API key no serviço externo.
    Implementação específica ao serviço — ex.: chamada REST para rotacionar.
    """
    import urllib.request, urllib.error
    # Exemplo: POST /rotate-key com Basic Auth usando old_api_key
    # request = urllib.request.Request(
    #     f"{endpoint}/rotate-key",
    #     data=json.dumps({"new_key": new_api_key}).encode(),
    #     headers={"Authorization": f"Bearer {old_api_key}"},
    #     method="POST",
    # )
    # urllib.request.urlopen(request)
    pass  # implementar conforme o serviço


def _test_credential(secret: dict) -> bool:
    """Testa a credencial AWSPENDING contra o serviço."""
    # Exemplo: GET /health com a nova API key
    # Deve retornar True se a credencial funciona, False caso contrário
    return True  # implementar conforme o serviço
```

---

## 7. Python — Aplicação Resiliente a Rotação

```python
"""
Padrão de aplicação resiliente a rotação de segredos.
Princípio: cache local com TTL + retry automático em falha de auth.
"""
import json
import time
import logging
import functools
from dataclasses import dataclass, field
from typing import Any, Optional
import boto3
import psycopg2
from psycopg2 import OperationalError

logger = logging.getLogger(__name__)

@dataclass
class CachedSecret:
    value: dict
    fetched_at: float
    ttl_seconds: int = 300  # 5 minutos — bem menor que o período de rotação (30 dias)

    def is_expired(self) -> bool:
        return time.time() - self.fetched_at > self.ttl_seconds


class SecretCache:
    """
    Cache de segredos com TTL.
    NÃO chame get_secret_value em cada request — use o cache.
    Cache TTL << período de rotação para que novas versões sejam descobertas.
    """
    def __init__(self, ttl_seconds: int = 300):
        self._cache: dict[str, CachedSecret] = {}
        self._ttl = ttl_seconds
        self._client = boto3.client("secretsmanager")

    def get(self, secret_name: str, force_refresh: bool = False) -> dict:
        cached = self._cache.get(secret_name)
        if cached and not cached.is_expired() and not force_refresh:
            return cached.value

        logger.info("Fetching secret %s from Secrets Manager.", secret_name)
        response = self._client.get_secret_value(SecretId=secret_name)
        value = json.loads(response["SecretString"])

        self._cache[secret_name] = CachedSecret(
            value=value,
            fetched_at=time.time(),
            ttl_seconds=self._ttl,
        )
        return value

    def invalidate(self, secret_name: str):
        self._cache.pop(secret_name, None)


# Instância global — compartilhada dentro da execução Lambda
_secret_cache = SecretCache(ttl_seconds=300)


def get_db_connection(secret_name: str, force_refresh: bool = False):
    """
    Obtém conexão PostgreSQL com credenciais do Secrets Manager.
    Se force_refresh=True, busca nova versão (após falha de auth).
    """
    secret = _secret_cache.get(secret_name, force_refresh=force_refresh)
    return psycopg2.connect(
        host=secret["host"],
        port=int(secret.get("port", 5432)),
        database=secret["dbname"],
        user=secret["username"],
        password=secret["password"],
        connect_timeout=5,
    )


def with_db_rotation_resilience(secret_name: str, max_retries: int = 2):
    """
    Decorator que adiciona resiliência a rotação de segredos.
    Em caso de falha de autenticação (OperationalError com "password authentication"),
    invalida o cache, busca nova versão e tenta novamente.
    
    Uso:
        @with_db_rotation_resilience("rds/postgres/app-user")
        def minha_funcao_db(conn, ...):
            ...
    """
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            last_error = None
            for attempt in range(max_retries + 1):
                conn = None
                try:
                    force_refresh = (attempt > 0)
                    if force_refresh:
                        logger.warning(
                            "Auth failure on attempt %d for %s. Refreshing secret.",
                            attempt, secret_name
                        )
                        _secret_cache.invalidate(secret_name)

                    conn = get_db_connection(secret_name, force_refresh=force_refresh)
                    return func(conn, *args, **kwargs)

                except OperationalError as e:
                    last_error = e
                    error_msg = str(e).lower()
                    # Só fazer retry se for erro de autenticação (possível rotação)
                    if "password authentication" not in error_msg \
                       and "authentication failed" not in error_msg:
                        raise  # erro de rede/DB real — não é rotação
                    logger.warning("Auth error (attempt %d/%d): %s", attempt+1, max_retries+1, e)
                    time.sleep(0.5 * (attempt + 1))  # backoff simples

                finally:
                    if conn:
                        conn.close()

            raise RuntimeError(
                f"Failed to connect after {max_retries + 1} attempts. "
                f"Last error: {last_error}"
            )
        return wrapper
    return decorator


# Exemplo de uso do decorator
@with_db_rotation_resilience("rds/postgres/app-user")
def create_order(conn, order_data: dict) -> int:
    """Cria um pedido — automaticamente resiliente a rotação."""
    with conn.cursor() as cur:
        cur.execute(
            "INSERT INTO orders (customer_id, total) VALUES (%s, %s) RETURNING id",
            (order_data["customer_id"], order_data["total"])
        )
        conn.commit()
        return cur.fetchone()[0]


# Lambda handler resiliente
def lambda_handler(event: dict, context) -> dict:
    try:
        order_id = create_order(order_data=event)
        return {"statusCode": 200, "orderId": order_id}
    except Exception as e:
        logger.error("Failed to process order: %s", e)
        return {"statusCode": 500, "error": str(e)}
```

---

## 8. CLI — Exemplos Essenciais

```bash
# 1. Criar segredo com geração automática de senha
aws secretsmanager create-secret \
  --name "rds/postgres/app-user" \
  --description "Credenciais da aplicação de checkout" \
  --secret-string '{
    "username": "app_user",
    "password": "WILL_BE_ROTATED",
    "host": "my-db.cluster-xxx.us-east-1.rds.amazonaws.com",
    "port": "5432",
    "dbname": "appdb",
    "engine": "postgres"
  }' \
  --kms-key-id "arn:aws:kms:us-east-1:123456789012:key/abc-123"

# 2. Ver versões e staging labels de um segredo
aws secretsmanager describe-secret \
  --secret-id "rds/postgres/app-user" \
  --query '{
    Name: Name,
    RotationEnabled: RotationEnabled,
    RotationRules: RotationRules,
    VersionIdsToStages: VersionIdsToStages
  }'

# 3. Buscar segredo atual (AWSCURRENT)
aws secretsmanager get-secret-value \
  --secret-id "rds/postgres/app-user" \
  --query 'SecretString' --output text | python3 -m json.tool

# 4. Buscar versão anterior (AWSPREVIOUS) — útil para rollback diagnóstico
aws secretsmanager get-secret-value \
  --secret-id "rds/postgres/app-user" \
  --version-stage AWSPREVIOUS \
  --query 'SecretString' --output text

# 5. Ativar rotação automática com Lambda nativa (30 dias)
aws secretsmanager rotate-secret \
  --secret-id "rds/postgres/app-user" \
  --rotation-lambda-arn "arn:aws:lambda:us-east-1:123456789012:function:SecretsManagerRDSPostgreSQLRotation" \
  --rotation-rules AutomaticallyAfterDays=30

# 6. Disparar rotação imediata (independente do schedule)
aws secretsmanager rotate-secret \
  --secret-id "rds/postgres/app-user"

# 7. Verificar status após rotação (checar se AWSPENDING foi resolvido)
aws secretsmanager list-secret-version-ids \
  --secret-id "rds/postgres/app-user" \
  --query 'Versions[*].{VersionId:VersionId,Stages:VersionStages,Date:LastAccessedDate}' \
  --output table

# 8. Cancelar rotação em andamento (se AWSPENDING ficou preso)
aws secretsmanager cancel-rotate-secret \
  --secret-id "rds/postgres/app-user"

# 9. Desabilitar rotação automática
aws secretsmanager rotate-secret \
  --secret-id "rds/postgres/app-user" \
  --no-rotate-immediately \
  --rotation-rules AutomaticallyAfterDays=0

# 10. Verificar logs de rotação (Lambda CloudWatch)
aws logs filter-log-events \
  --log-group-name "/aws/lambda/SecretsManagerRDSPostgreSQLRotation" \
  --start-time $(date -d '1 hour ago' +%s000) \
  --filter-pattern "ERROR" \
  --query 'events[*].message' --output text

# 11. Listar todos os segredos com rotação habilitada
aws secretsmanager list-secrets \
  --filters Key=rotation-enabled,Values=true \
  --query 'SecretList[*].{Name:Name,RotationEnabled:RotationEnabled,NextRotation:NextRotationDate}' \
  --output table

# 12. Resource policy — permitir acesso cross-account ao segredo
aws secretsmanager put-resource-policy \
  --secret-id "rds/postgres/app-user" \
  --resource-policy '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::987654321098:role/AppRole"},
      "Action": ["secretsmanager:GetSecretValue", "secretsmanager:DescribeSecret"],
      "Resource": "*"
    }]
  }'
```

---

## 9. Diagrama: Ciclo Completo de Rotação

```
          CICLO DE ROTAÇÃO — SINGLE USER (30 dias)

 t=0 (schedule disparado)
  │
  ├─ createSecret ─────────────────────────────────────────────────
  │   put_secret_value(token, AWSPENDING)
  │   → nova senha gerada, armazenada com label AWSPENDING
  │
  ├─ setSecret ────────────────────────────────────────────────────
  │   ALTER USER app_user PASSWORD 'nova_senha_do_AWSPENDING'
  │   ↑ banco tem nova senha; AWSCURRENT ainda tem a antiga
  │   ← JANELA DE RISCO (milissegundos a segundos) →
  │
  ├─ testSecret ───────────────────────────────────────────────────
  │   SELECT 1 usando AWSPENDING → OK
  │
  └─ finishSecret ─────────────────────────────────────────────────
      update_secret_version_stage:
        AWSCURRENT  → versão antiga = AWSPREVIOUS
        AWSPENDING  → nova versão  = AWSCURRENT (+ AWSPENDING removido)
  
  ↓ após finishSecret
  Aplicação com cache TTL expirado busca AWSCURRENT → nova senha
  Aplicação com cache ainda válido: retenta com nova senha se auth falhar

  ↓ t+30 dias: próxima rotação
```

---

## 10. Armadilhas (Pitfalls)

[FATO] **Lambda sem acesso à rede = rotação falha imediatamente**: a Lambda de rotação precisa alcançar tanto o endpoint do Secrets Manager quanto o banco. Sem VPC Endpoint ou NAT Gateway, a rotação falha com `SecretsManagerServiceException`. Verificar: subnet privada, route table, security groups.

[FATO] **AWSPENDING "preso" em versão vazia = rotação futura bloqueada**: se uma rotação falhou entre createSecret e setSecret, `AWSPENDING` pode estar anexado a uma versão com conteúdo vazio. Toda rotação subsequente retorna erro assumindo que há rotação em andamento. Solução: `cancel-rotate-secret` para limpar o AWSPENDING órfão.

[FATO] **Alternating Users incompatível com RDS Proxy**: a documentação AWS é explícita — RDS Proxy não suporta a estratégia Alternating Users. Use Single User quando tiver RDS Proxy na frente.

[FATO] **Não logar SecretString**: o Lambda de rotação tem acesso a `SecretString` em plaintext. Um `logger.debug(response)` acidental expõe a senha em CloudWatch Logs. A documentação AWS alerta explicitamente sobre isso.

[CONSENSO] **Cache muito longo aumenta janela de risco**: se a aplicação cacheia a senha por 24 horas e a senha foi rotacionada, há 24 horas de falhas de auth antes do cache expirar. TTL de 5 minutos + retry em falha de auth é o padrão seguro.

[FATO] **Permissão mínima na execution role da Lambda**: a Lambda de rotação não deve ter `secretsmanager:*`. As permissões mínimas são: `GetSecretValue`, `PutSecretValue`, `UpdateSecretVersionStage`, `DescribeSecret` para o(s) segredo(s) específico(s), mais `kms:Decrypt`/`kms:GenerateDataKey` se usar CMK.

[FATO] **Rotação mínima a cada 4 horas**: Secrets Manager permite schedule com `rate(4 hours)` como mínimo. Rotações mais frequentes são rejeitadas.

[CONSENSO] **`secretsmanager:SecretId` na Lambda resource policy**: para evitar confused deputy attack, adicione `aws:SourceAccount` na resource policy da Lambda de rotação — impede que outras contas invoquem a Lambda fingindo ser o Secrets Manager.

---

## 11. Quando Usar Cada Estratégia

```
╔══════════════════════════════╦═══════════════════════════════════════╗
║ Cenário                      ║ Estratégia recomendada                ║
╠══════════════════════════════╬═══════════════════════════════════════╣
║ Aplicação geral (maioria)    ║ Single User + retry com backoff       ║
║ Com RDS Proxy                ║ Single User (única opção suportada)   ║
║ Alta disponibilidade crítica ║ Alternating Users (zero-downtime)     ║
║ API key de serviço externo   ║ Rotation function customizada         ║
║ Usuário interativo / ad hoc  ║ Single User (whitepaper recomenda)    ║
║ Banco sem suporte a CLONE    ║ Single User                           ║
╚══════════════════════════════╩═══════════════════════════════════════╝
```

---

## Exercício de Reflexão

Uma aplicação Lambda de alto tráfego (10.000 req/s) se conecta a um RDS PostgreSQL via RDS Proxy. Atualmente usa credenciais hard-coded no código. O time quer migrar para Secrets Manager com rotação automática a cada 30 dias, sem qualquer downtime.

**Desenhe a arquitetura e responda:**

1. Qual estratégia de rotação escolher e por quê? (hint: RDS Proxy)
2. Como a aplicação deve buscar e cachear o segredo de forma segura? Qual TTL faz sentido?
3. Durante os ~2 segundos da janela de risco do setSecret, o que acontece com as 10.000 req/s em andamento?
4. Como o `with_db_rotation_resilience` decorator resolve o problema? Qual é o fluxo exato de retry?
5. A Lambda de rotação está em subnet privada sem NAT Gateway nem VPC Endpoint. O que acontece e como corrigir?

---

## Referências

- [FATO] [Lambda rotation functions](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotate-secrets_lambda-functions.html) — docs.aws.amazon.com
- [FATO] [Lambda function rotation strategies](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotation-strategy.html) — docs.aws.amazon.com
- [FATO] [Set up automatic rotation for Amazon RDS](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotate-secrets_turn-on-for-db.html) — docs.aws.amazon.com
- [FATO] [Troubleshoot rotation](https://docs.aws.amazon.com/secretsmanager/latest/userguide/troubleshoot_rotation.html) — docs.aws.amazon.com
- [FATO] [Lambda rotation function templates](https://docs.aws.amazon.com/secretsmanager/latest/userguide/reference_available-rotation-templates.html) — docs.aws.amazon.com
