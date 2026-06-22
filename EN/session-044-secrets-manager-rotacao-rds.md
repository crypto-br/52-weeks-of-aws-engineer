# Session 044 — Secrets Manager: Automatic Rotation, Lambda Rotators, and RDS Integration

**Estimated duration:** 60 min  
**Prerequisite:** none (self-contained session)

---

## Session Objectives

- Understand the version lifecycle and staging labels (AWSPENDING → AWSCURRENT → AWSPREVIOUS)
- Implement the four steps of the rotation function (createSecret, setSecret, testSecret, finishSecret)
- Distinguish Single User vs Alternating Users and when each strategy is appropriate
- Configure native rotation for RDS PostgreSQL with AWS-managed Lambda
- Build a custom rotator for an external service
- Write applications resilient to rotation (cache + retry without downtime)

---

## 1. Secrets Manager Fundamentals

### 1.1 Versions and Staging Labels

[FACT] Each secret in Secrets Manager can have **multiple simultaneous versions**. Each version receives one or more **staging labels** that identify its role in the rotation cycle.

```
Staging label states during rotation:

  ┌──────────────────────────────────────────────────────────────┐
  │ Before rotation                                              │
  │   version A: [AWSCURRENT]                                     │
  │   version B: [AWSPREVIOUS]  (from previous rotation)         │
  └──────────────────────────────────────────────────────────────┘
  
  ┌──────────────────────────────────────────────────────────────┐
  │ During rotation (between createSecret and finishSecret)      │
  │   version A: [AWSCURRENT]                                     │
  │   version B: [AWSPREVIOUS]                                    │
  │   version C: [AWSPENDING]   ← new password generated, DB updated│
  └──────────────────────────────────────────────────────────────┘
  
  ┌──────────────────────────────────────────────────────────────┐
  │ After finishSecret (rotation complete)                       │
  │   version A: [AWSPREVIOUS]  ← was AWSCURRENT                 │
  │   version B: no label       ← automatically removed           │
  │   version C: [AWSCURRENT]   ← new version promoted           │
  └──────────────────────────────────────────────────────────────┘
```

[FACT] `AWSPENDING` should never be manually removed before `finishSecret`. If `AWSPENDING` exists without being on the same versionId as `AWSCURRENT`, any new rotation attempt will assume a previous rotation is in progress and return an error.

### 1.2 Encryption

[FACT] All secrets are encrypted at rest. By default, it uses the AWS-managed key `aws/secretsmanager` (no additional KMS cost, but no key policy control). For granular control, use a CMK (Customer Managed Key).

[FACT] If a secret uses a custom CMK, the rotation Lambda needs `kms:Decrypt` and `kms:GenerateDataKey` permissions on the CMK. Use `kms:EncryptionContext:SecretARN` to limit the Lambda's access to only the secret it rotates.

### 1.3 Pricing

[FACT] $0.40/secret/month + $0.05 per 10,000 API calls. The first 30 days of a new secret are free.

---

## 2. The Four Steps of the Rotation Function

[FACT] Every rotation function — native or custom — must implement four methods invoked in sequence by Secrets Manager:

```
Sequential rotation flow:

  Secrets Manager
       │
       ├──1──► createSecret  → generates new password
       │                     → put_secret_value with AWSPENDING label
       │                     → idempotent: checks if AWSPENDING already exists
       │
       ├──2──► setSecret     → applies new password to target database/service
       │                     → uses AWSCURRENT credentials to connect
       │                     → changes user password to the AWSPENDING value
       │
       ├──3──► testSecret    → tests connection using AWSPENDING
       │                     → reads something from DB to confirm it works
       │                     → if it fails → rotation stops; AWSPENDING remains
       │
       └──4──► finishSecret  → update_secret_version_stage:
                               moves AWSCURRENT → old version becomes AWSPREVIOUS
                               AWSPENDING version → becomes new AWSCURRENT
                               AWSPENDING is removed atomically
```

### 2.1 Idempotency and ClientRequestToken

[FACT] Secrets Manager passes a `ClientRequestToken` (UUID) as `VersionId` for each rotation call. `createSecret` must check if a version with that token already exists before generating a new password — this ensures idempotency if the Lambda is called again after a partial failure.

---

## 3. Rotation Strategies

### 3.1 Single User (recommended for most cases)

[FACT] Updates credentials for **a single user** in a single secret. The sequence is: generate new password → apply to database → update secret.

```
Timeline — Single User:

  t0  createSecret:  AWSPENDING created with new password
  t1  setSecret:     DB changes user password
  t2  [risk window]: DB already has new password, AWSCURRENT still has the old one
                     → new connections with AWSCURRENT fail
  t3  testSecret:    tests AWSPENDING → success
  t4  finishSecret:  AWSPENDING promoted to AWSCURRENT
  t5  [normalized]:  all new connections use new password

Risk window duration (t1→t4): seconds to milliseconds
Mitigation: exponential backoff + retry in the application
```

[FACT] Single User is recommended for: general cases, ad hoc/interactive users, and when the database does not support cloning users. **RDS Proxy supports Single User.**

### 3.2 Alternating Users (high availability)

[FACT] Maintains two users (e.g.: `myuser` and `myuser_clone`) and alternates which one gets the password updated. Rotation works as follows:

```
First rotation:
  AWSPENDING = {username: "myuser_clone", password: generated}
  Lambda clones "myuser" → creates "myuser_clone" with same permissions
  AWSCURRENT now points to myuser_clone

Second rotation:
  AWSPENDING = {username: "myuser", password: new_password}
  Lambda updates "myuser" password (no cloning needed)
  AWSCURRENT now points to myuser

Third rotation: same as first (alternates back to myuser_clone)
```

[FACT] Alternating Users requires a **second secret** with superuser credentials (which has permission to create/clone users in the database). The Lambda uses the superuser to clone and modify permissions.

[FACT] **Amazon RDS Proxy does NOT support Alternating Users** — use Single User when you have RDS Proxy.

[FACT] If the original user's permissions are modified after the clone is created, the cloned user **is not automatically updated** — manual update is required.

---

## 4. Network Access — Critical Requirement

[FACT] The rotation Lambda needs to reach **two endpoints simultaneously**:

```
Rotation Lambda (in VPC)
    │
    ├──► Secrets Manager endpoint
    │    Option A: VPC Endpoint (Interface) for secretsmanager — recommended
    │    Option B: NAT Gateway + Internet Gateway
    │
    └──► RDS PostgreSQL endpoint (port 5432)
         Security Group: allow Lambda SG → RDS SG on port 5432
```

[FACT] If the Lambda cannot reach the Secrets Manager endpoint, rotation fails immediately with `SecretsManagerServiceException`. The most common cause of rotation failure is a network/VPC issue.

---

## 5. CDK Python — Native RDS PostgreSQL Rotation

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

        # KMS CMK for secret encryption
        secret_key = kms.Key(self, "SecretKey",
            description="CMK for Secrets Manager - RDS credentials",
            enable_key_rotation=True,  # annual rotation of the CMK itself
        )

        # ──────────────────────────────────────────────────────────────
        # RDS superuser secret (admin) — required for
        # Alternating Users strategy
        # ──────────────────────────────────────────────────────────────
        superuser_secret = sm.Secret(self, "RDSSuperuserSecret",
            secret_name="rds/postgres/superuser",
            description="RDS PostgreSQL superuser credentials",
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
        # RDS PostgreSQL instance with credential rotation
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
        # Application secret (user "app_user") — will be rotated
        # with Alternating Users strategy
        # ──────────────────────────────────────────────────────────────
        app_secret = sm.Secret(self, "AppUserSecret",
            secret_name="rds/postgres/app-user",
            description="Application user credentials — automatic rotation",
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
        # Automatic rotation: Alternating Users strategy
        # HostedRotationLambda = AWS-managed Lambda (native rotation)
        # ──────────────────────────────────────────────────────────────
        app_secret.add_rotation_schedule("RotationSchedule",
            hosted_rotation=sm.HostedRotation.postgre_sql_multi_user(
                function_name="SecretsManagerRDSPostgreSQLRotation",
                master_secret=superuser_secret,  # superuser to clone
                vpc=vpc,
                vpc_subnets=ec2.SubnetSelection(
                    subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS
                ),
                security_groups=[ec2.SecurityGroup(self, "RotationLambdaSG", vpc=vpc)],
                exclude_characters="/@\"\\' ;",
            ),
            automatically_after=Duration.days(30),  # rotate every 30 days
        )

        # Allow rotation Lambda to connect to RDS
        db_sg.add_ingress_rule(
            peer=ec2.Peer.ipv4(vpc.vpc_cidr_block),
            connection=ec2.Port.tcp(5432),
            description="Allow rotation Lambda to access PostgreSQL",
        )

        # VPC Endpoint for Secrets Manager — Lambda doesn't need NAT
        vpc.add_interface_endpoint("SecretsManagerEndpoint",
            service=ec2.InterfaceVpcEndpointAwsService.SECRETS_MANAGER,
        )

        # ──────────────────────────────────────────────────────────────
        # Single User rotation example (simpler)
        # For when Alternating Users is not needed
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

## 6. Python — Custom Rotation Function (External Service)

```python
"""
Custom rotation function template for external services
(e.g.: third-party API key, OAuth token, legacy system password).

Deployed via CDK Lambda Function pointed as rotation function on the secret.
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
    Entry point: Secrets Manager invokes this function with an event
    containing arn, token, and step.
    """
    arn   = event["SecretId"]
    token = event["ClientRequestToken"]
    step  = event["Step"]

    # Verify that the secret has the version with this token
    metadata = secretsmanager.describe_secret(SecretId=arn)
    versions = metadata.get("VersionIdsToStages", {})

    if token not in versions:
        raise ValueError(f"Secret version {token} has no stage for secret {arn}")

    if "AWSCURRENT" in versions[token]:
        # Rotation already completed for this token — idempotent
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
    Step 1: Generates new credential and stores as AWSPENDING.
    Idempotent: if AWSPENDING already exists with this token, does nothing.
    """
    # Try to fetch the existing AWSPENDING version (for idempotency)
    try:
        client.get_secret_value(SecretId=arn, VersionId=token, VersionStage="AWSPENDING")
        logger.info("createSecret: AWSPENDING version %s already exists. Idempotent skip.", token)
        return
    except client.exceptions.ResourceNotFoundException:
        pass  # AWSPENDING does not exist yet — proceed

    # Fetch current secret to get structure
    current = json.loads(
        client.get_secret_value(SecretId=arn, VersionStage="AWSCURRENT")["SecretString"]
    )

    # Generate new credential (e.g.: new API key)
    new_credential = _generate_new_credential(current)

    # Store as AWSPENDING
    client.put_secret_value(
        SecretId=arn,
        ClientRequestToken=token,
        SecretString=json.dumps(new_credential),
        VersionStages=["AWSPENDING"],
    )
    logger.info("createSecret: new version stored as AWSPENDING for %s.", arn)


def set_secret(client, arn: str, token: str) -> None:
    """
    Step 2: Applies the new credential to the target service.
    IMPORTANT: verify that AWSCURRENT and AWSPENDING point to the same resource
    before modifying (defense against confused deputy).
    """
    current_secret = json.loads(
        client.get_secret_value(SecretId=arn, VersionStage="AWSCURRENT")["SecretString"]
    )
    pending_secret = json.loads(
        client.get_secret_value(SecretId=arn, VersionId=token, VersionStage="AWSPENDING")["SecretString"]
    )

    # Security validation: same target resource
    if current_secret.get("endpoint") != pending_secret.get("endpoint"):
        raise ValueError("AWSCURRENT and AWSPENDING point to different endpoints. Aborting.")

    # Apply the new credential to the external service
    _apply_credential_to_service(
        endpoint=pending_secret["endpoint"],
        old_api_key=current_secret["api_key"],
        new_api_key=pending_secret["api_key"],
    )
    logger.info("setSecret: credential updated in service for %s.", arn)


def test_secret(client, arn: str, token: str) -> None:
    """
    Step 3: Validates that the new credential (AWSPENDING) works.
    """
    pending_secret = json.loads(
        client.get_secret_value(SecretId=arn, VersionId=token, VersionStage="AWSPENDING")["SecretString"]
    )

    # Test the credential against the service
    if not _test_credential(pending_secret):
        raise RuntimeError(
            f"testSecret: new credential failed validation for {arn}. "
            "Rotation will not complete."
        )
    logger.info("testSecret: new credential validated successfully for %s.", arn)


def finish_secret(client, arn: str, token: str) -> None:
    """
    Step 4: Promotes AWSPENDING to AWSCURRENT atomically.
    Secrets Manager automatically moves the previous version to AWSPREVIOUS.
    DO NOT remove AWSPENDING manually before this call.
    """
    # Find the current versionId of AWSCURRENT
    metadata = client.describe_secret(SecretId=arn)
    current_version = next(
        (vid for vid, stages in metadata["VersionIdsToStages"].items()
         if "AWSCURRENT" in stages and vid != token),
        None,
    )

    # Move AWSCURRENT from old version to new version (token)
    # This call also removes AWSPENDING atomically
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


# ── External service helper functions ──────────────────────────────────

def _generate_new_credential(current: dict) -> dict:
    """Generates new credential maintaining the secret structure."""
    import secrets
    return {
        **current,
        "api_key": secrets.token_urlsafe(32),
        "generated_at": __import__("datetime").datetime.utcnow().isoformat() + "Z",
    }


def _apply_credential_to_service(endpoint: str, old_api_key: str, new_api_key: str):
    """
    Applies the new API key to the external service.
    Service-specific implementation — e.g.: REST call to rotate.
    """
    import urllib.request, urllib.error
    # Example: POST /rotate-key with Basic Auth using old_api_key
    # request = urllib.request.Request(
    #     f"{endpoint}/rotate-key",
    #     data=json.dumps({"new_key": new_api_key}).encode(),
    #     headers={"Authorization": f"Bearer {old_api_key}"},
    #     method="POST",
    # )
    # urllib.request.urlopen(request)
    pass  # implement according to the service


def _test_credential(secret: dict) -> bool:
    """Tests the AWSPENDING credential against the service."""
    # Example: GET /health with the new API key
    # Should return True if credential works, False otherwise
    return True  # implement according to the service
```

---

## 7. Python — Application Resilient to Rotation

```python
"""
Application pattern resilient to secret rotation.
Principle: local cache with TTL + automatic retry on auth failure.
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
    ttl_seconds: int = 300  # 5 minutes — much shorter than rotation period (30 days)

    def is_expired(self) -> bool:
        return time.time() - self.fetched_at > self.ttl_seconds


class SecretCache:
    """
    Secret cache with TTL.
    DO NOT call get_secret_value on each request — use the cache.
    Cache TTL << rotation period so new versions are discovered.
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


# Global instance — shared within Lambda execution
_secret_cache = SecretCache(ttl_seconds=300)


def get_db_connection(secret_name: str, force_refresh: bool = False):
    """
    Gets PostgreSQL connection with Secrets Manager credentials.
    If force_refresh=True, fetches new version (after auth failure).
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
    Decorator that adds resilience to secret rotation.
    On authentication failure (OperationalError with "password authentication"),
    invalidates the cache, fetches new version, and retries.
    
    Usage:
        @with_db_rotation_resilience("rds/postgres/app-user")
        def my_db_function(conn, ...):
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
                    # Only retry if it's an authentication error (possible rotation)
                    if "password authentication" not in error_msg \
                       and "authentication failed" not in error_msg:
                        raise  # real network/DB error — not rotation
                    logger.warning("Auth error (attempt %d/%d): %s", attempt+1, max_retries+1, e)
                    time.sleep(0.5 * (attempt + 1))  # simple backoff

                finally:
                    if conn:
                        conn.close()

            raise RuntimeError(
                f"Failed to connect after {max_retries + 1} attempts. "
                f"Last error: {last_error}"
            )
        return wrapper
    return decorator


# Example usage of the decorator
@with_db_rotation_resilience("rds/postgres/app-user")
def create_order(conn, order_data: dict) -> int:
    """Creates an order — automatically resilient to rotation."""
    with conn.cursor() as cur:
        cur.execute(
            "INSERT INTO orders (customer_id, total) VALUES (%s, %s) RETURNING id",
            (order_data["customer_id"], order_data["total"])
        )
        conn.commit()
        return cur.fetchone()[0]


# Resilient Lambda handler
def lambda_handler(event: dict, context) -> dict:
    try:
        order_id = create_order(order_data=event)
        return {"statusCode": 200, "orderId": order_id}
    except Exception as e:
        logger.error("Failed to process order: %s", e)
        return {"statusCode": 500, "error": str(e)}
```

---

## 8. CLI — Essential Examples

```bash
# 1. Create secret with automatic password generation
aws secretsmanager create-secret \
  --name "rds/postgres/app-user" \
  --description "Checkout application credentials" \
  --secret-string '{
    "username": "app_user",
    "password": "WILL_BE_ROTATED",
    "host": "my-db.cluster-xxx.us-east-1.rds.amazonaws.com",
    "port": "5432",
    "dbname": "appdb",
    "engine": "postgres"
  }' \
  --kms-key-id "arn:aws:kms:us-east-1:123456789012:key/abc-123"

# 2. View versions and staging labels of a secret
aws secretsmanager describe-secret \
  --secret-id "rds/postgres/app-user" \
  --query '{
    Name: Name,
    RotationEnabled: RotationEnabled,
    RotationRules: RotationRules,
    VersionIdsToStages: VersionIdsToStages
  }'

# 3. Fetch current secret (AWSCURRENT)
aws secretsmanager get-secret-value \
  --secret-id "rds/postgres/app-user" \
  --query 'SecretString' --output text | python3 -m json.tool

# 4. Fetch previous version (AWSPREVIOUS) — useful for rollback diagnostics
aws secretsmanager get-secret-value \
  --secret-id "rds/postgres/app-user" \
  --version-stage AWSPREVIOUS \
  --query 'SecretString' --output text

# 5. Enable automatic rotation with native Lambda (30 days)
aws secretsmanager rotate-secret \
  --secret-id "rds/postgres/app-user" \
  --rotation-lambda-arn "arn:aws:lambda:us-east-1:123456789012:function:SecretsManagerRDSPostgreSQLRotation" \
  --rotation-rules AutomaticallyAfterDays=30

# 6. Trigger immediate rotation (regardless of schedule)
aws secretsmanager rotate-secret \
  --secret-id "rds/postgres/app-user"

# 7. Check status after rotation (verify AWSPENDING was resolved)
aws secretsmanager list-secret-version-ids \
  --secret-id "rds/postgres/app-user" \
  --query 'Versions[*].{VersionId:VersionId,Stages:VersionStages,Date:LastAccessedDate}' \
  --output table

# 8. Cancel rotation in progress (if AWSPENDING got stuck)
aws secretsmanager cancel-rotate-secret \
  --secret-id "rds/postgres/app-user"

# 9. Disable automatic rotation
aws secretsmanager rotate-secret \
  --secret-id "rds/postgres/app-user" \
  --no-rotate-immediately \
  --rotation-rules AutomaticallyAfterDays=0

# 10. Check rotation logs (Lambda CloudWatch)
aws logs filter-log-events \
  --log-group-name "/aws/lambda/SecretsManagerRDSPostgreSQLRotation" \
  --start-time $(date -d '1 hour ago' +%s000) \
  --filter-pattern "ERROR" \
  --query 'events[*].message' --output text

# 11. List all secrets with rotation enabled
aws secretsmanager list-secrets \
  --filters Key=rotation-enabled,Values=true \
  --query 'SecretList[*].{Name:Name,RotationEnabled:RotationEnabled,NextRotation:NextRotationDate}' \
  --output table

# 12. Resource policy — allow cross-account access to secret
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

## 9. Diagram: Complete Rotation Cycle

```
          ROTATION CYCLE — SINGLE USER (30 days)

 t=0 (schedule triggered)
  │
  ├─ createSecret ─────────────────────────────────────────────────
  │   put_secret_value(token, AWSPENDING)
  │   → new password generated, stored with AWSPENDING label
  │
  ├─ setSecret ────────────────────────────────────────────────────
  │   ALTER USER app_user PASSWORD 'new_password_from_AWSPENDING'
  │   ↑ DB has new password; AWSCURRENT still has the old one
  │   ← RISK WINDOW (milliseconds to seconds) →
  │
  ├─ testSecret ───────────────────────────────────────────────────
  │   SELECT 1 using AWSPENDING → OK
  │
  └─ finishSecret ─────────────────────────────────────────────────
      update_secret_version_stage:
        AWSCURRENT  → old version = AWSPREVIOUS
        AWSPENDING  → new version  = AWSCURRENT (+ AWSPENDING removed)
  
  ↓ after finishSecret
  Application with expired cache TTL fetches AWSCURRENT → new password
  Application with still-valid cache: retries with new password if auth fails

  ↓ t+30 days: next rotation
```

---

## 10. Pitfalls

[FACT] **Lambda without network access = rotation fails immediately**: the rotation Lambda needs to reach both the Secrets Manager endpoint and the database. Without a VPC Endpoint or NAT Gateway, rotation fails with `SecretsManagerServiceException`. Check: private subnet, route table, security groups.

[FACT] **AWSPENDING "stuck" on empty version = future rotation blocked**: if a rotation failed between createSecret and setSecret, `AWSPENDING` may be attached to a version with empty content. Every subsequent rotation returns an error assuming there is a rotation in progress. Solution: `cancel-rotate-secret` to clean the orphaned AWSPENDING.

[FACT] **Alternating Users incompatible with RDS Proxy**: AWS documentation is explicit — RDS Proxy does not support the Alternating Users strategy. Use Single User when you have RDS Proxy in front.

[FACT] **Do not log SecretString**: the rotation Lambda has access to `SecretString` in plaintext. An accidental `logger.debug(response)` exposes the password in CloudWatch Logs. AWS documentation explicitly warns about this.

[CONSENSUS] **Cache that is too long increases risk window**: if the application caches the password for 24 hours and the password was rotated, there are 24 hours of auth failures before the cache expires. 5-minute TTL + retry on auth failure is the secure standard.

[FACT] **Minimum permissions on the Lambda execution role**: the rotation Lambda should not have `secretsmanager:*`. Minimum permissions are: `GetSecretValue`, `PutSecretValue`, `UpdateSecretVersionStage`, `DescribeSecret` for the specific secret(s), plus `kms:Decrypt`/`kms:GenerateDataKey` if using CMK.

[FACT] **Minimum rotation interval is 4 hours**: Secrets Manager allows schedule with `rate(4 hours)` as minimum. More frequent rotations are rejected.

[CONSENSUS] **`secretsmanager:SecretId` in Lambda resource policy**: to prevent confused deputy attacks, add `aws:SourceAccount` in the rotation Lambda's resource policy — prevents other accounts from invoking the Lambda pretending to be Secrets Manager.

---

## 11. When to Use Each Strategy

```
╔══════════════════════════════╦═══════════════════════════════════════╗
║ Scenario                     ║ Recommended strategy                  ║
╠══════════════════════════════╬═══════════════════════════════════════╣
║ General application (most)   ║ Single User + retry with backoff      ║
║ With RDS Proxy               ║ Single User (only supported option)   ║
║ Critical high availability   ║ Alternating Users (zero-downtime)     ║
║ External service API key     ║ Custom rotation function              ║
║ Interactive / ad hoc user    ║ Single User (whitepaper recommends)   ║
║ DB without CLONE support     ║ Single User                           ║
╚══════════════════════════════╩═══════════════════════════════════════╝
```

---

## Reflection Exercise

A high-traffic Lambda application (10,000 req/s) connects to an RDS PostgreSQL via RDS Proxy. It currently uses hard-coded credentials in the code. The team wants to migrate to Secrets Manager with automatic rotation every 30 days, without any downtime.

**Design the architecture and answer:**

1. Which rotation strategy to choose and why? (hint: RDS Proxy)
2. How should the application fetch and cache the secret securely? What TTL makes sense?
3. During the ~2 seconds of the setSecret risk window, what happens with the 10,000 req/s in progress?
4. How does the `with_db_rotation_resilience` decorator solve the problem? What is the exact retry flow?
5. The rotation Lambda is in a private subnet without a NAT Gateway or VPC Endpoint. What happens and how to fix it?

---

## References

- [FACT] [Lambda rotation functions](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotate-secrets_lambda-functions.html) — docs.aws.amazon.com
- [FACT] [Lambda function rotation strategies](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotation-strategy.html) — docs.aws.amazon.com
- [FACT] [Set up automatic rotation for Amazon RDS](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotate-secrets_turn-on-for-db.html) — docs.aws.amazon.com
- [FACT] [Troubleshoot rotation](https://docs.aws.amazon.com/secretsmanager/latest/userguide/troubleshoot_rotation.html) — docs.aws.amazon.com
- [FACT] [Lambda rotation function templates](https://docs.aws.amazon.com/secretsmanager/latest/userguide/reference_available-rotation-templates.html) — docs.aws.amazon.com
