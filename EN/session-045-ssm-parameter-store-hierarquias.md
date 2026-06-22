# Session 045 — SSM Parameter Store: Hierarchies, SecureString, and Decision vs Secrets Manager

**Estimated duration:** 60 min  
**Prerequisite:** session-044 (Secrets Manager, rotation, staging labels)

---

## Session Objectives

- Create and retrieve parameters in hierarchy with `get-parameters-by-path` (recursive, filters)
- Understand the three types: String, StringList, SecureString (with default key vs CMK)
- Compare tiers Standard vs Advanced vs Intelligent-Tiering (cost, limits, policies)
- Use Parameter Policies (Expiration, ExpirationNotification, NoChangeNotification) — Advanced only
- Reference parameters in CDK/CloudFormation via dynamic references
- Document the decision criteria between SSM Parameter Store and Secrets Manager

---

## 1. Parameter Types

[FACT] Parameter Store supports three types:

```
╔═════════════════╦══════════════════════════════════════════════════════╗
║ Type            ║ Characteristics                                      ║
╠═════════════════╬══════════════════════════════════════════════════════╣
║ String          ║ Plaintext. Any text. E.g.: ARN, endpoint, flag       ║
╠═════════════════╬══════════════════════════════════════════════════════╣
║ StringList      ║ Comma-separated list of strings.                     ║
║                 ║ E.g.: "sg-abc,sg-def" (used with aws:ssm:type)       ║
╠═════════════════╬══════════════════════════════════════════════════════╣
║ SecureString    ║ Encrypted with KMS at rest and in transit.           ║
║                 ║ Default key aws/ssm or customer CMK.                 ║
╚═════════════════╩══════════════════════════════════════════════════════╝
```

### 1.1 SecureString — Default Key vs CMK

[FACT] SecureString with default key `aws/ssm`:
- Free (no additional KMS charges)
- AWS-managed key — no key policy control
- Does not allow cross-account decrypt
- One key per region per account — no per-parameter isolation

[FACT] SecureString with CMK (Customer Managed Key):
- KMS charge per encryption operation
- Granular control via key policy (who can encrypt/decrypt)
- Allows cross-account: account B can decrypt if it has permission in account A's key policy
- Enables audit trail via CloudTrail per key

[FACT] SecureString throughput is limited by the region's KMS throughput (5,500–10,000 TPS depending on region and key type).

---

## 2. Tiers: Standard, Advanced, Intelligent-Tiering

[FACT] Complete comparison table (AWS documentation):

```
╔═══════════════════════════════════╦═══════════════╦══════════════╗
║ Characteristic                    ║ Standard      ║ Advanced     ║
╠═══════════════════════════════════╬═══════════════╬══════════════╣
║ Parameters per region/account     ║ 10,000        ║ 100,000      ║
║ Maximum value size                ║ 4 KB          ║ 8 KB         ║
║ Parameter Policies                ║ No            ║ Yes          ║
║ Cross-account sharing             ║ No            ║ Yes          ║
║ Storage cost                      ║ Free          ║ $0.05/param/month ║
╚═══════════════════════════════════╩═══════════════╩══════════════╝
```

[FACT] Promotion from Standard → Advanced is **irreversible**: it is not possible to revert an Advanced parameter to Standard without deleting and recreating it (which causes data and version history loss). To save costs, you must delete and recreate as Standard.

[FACT] **Intelligent-Tiering**: Parameter Store evaluates each `PutParameter` and creates Standard automatically, except when: value > 4 KB, account already has ≥ 10,000 parameters, or parameter uses policies. In those cases, it promotes to Advanced. Recommended for teams with uncertain usage growth.

### 2.1 Throughput

[FACT] Parameter Store's default throughput is **40 TPS**. With high-throughput enabled (paid), it increases to up to **10,000 TPS**.

[FACT] The cost of high-throughput is $0.05 per 10,000 API calls (when enabled). If you only use 40 TPS or less, there is no reason to enable high-throughput.

---

## 3. Parameter Hierarchies

[FACT] Parameter Store supports hierarchical structure by path, with up to **15 levels** of depth separated by `/`. The full name (path) has a 2,048 character limit.

### 3.1 Recommended hierarchy convention

```
/application/environment/component/key

Practical examples:
  /checkout/prod/database/host
  /checkout/prod/database/port
  /checkout/prod/database/name
  /checkout/prod/database/password          ← SecureString
  /checkout/staging/database/host
  /checkout/staging/database/password       ← SecureString
  /checkout/prod/redis/endpoint
  /checkout/prod/feature-flags/new-checkout-enabled
  /shared/prod/observability/datadog-api-key ← SecureString
  /shared/prod/certificates/tls-cert         ← String (PEM)
```

### 3.2 GetParametersByPath — Retrieve an entire hierarchy

[FACT] `GetParametersByPath` returns parameters under a path prefix. With `--recursive`, it includes all sublevels. With `--with-decryption`, it decrypts SecureString automatically.

```
/checkout/prod/database/host     ←─┐
/checkout/prod/database/port     ←─┤ GetParametersByPath("/checkout/prod/database", recursive=False)
/checkout/prod/database/password ←─┘

/checkout/prod/database/host     ←─┐
/checkout/prod/database/port     ←─┤ GetParametersByPath("/checkout/prod", recursive=True)
/checkout/prod/database/password ←─┤
/checkout/prod/redis/endpoint    ←─┘
```

### 3.3 IAM — Scoping by hierarchy

[FACT] IAM permissions can be scoped by path prefix using the parameter ARN:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameter",
        "ssm:GetParametersByPath",
        "ssm:GetParameters"
      ],
      "Resource": [
        "arn:aws:ssm:us-east-1:123456789012:parameter/checkout/prod/*",
        "arn:aws:ssm:us-east-1:123456789012:parameter/shared/prod/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": ["kms:Decrypt"],
      "Resource": "arn:aws:kms:us-east-1:123456789012:key/abc-123"
    }
  ]
}
```

---

## 4. Parameter Policies (Advanced only)

[FACT] Parameter Policies allow associating automatic behaviors to an Advanced parameter. Three types:

```
╔═══════════════════════════════╦═════════════════════════════════════════╗
║ Policy Type                   ║ Behavior                                ║
╠═══════════════════════════════╬═════════════════════════════════════════╣
║ Expiration                    ║ Automatically deletes the parameter at  ║
║                               ║ a specific date/time                    ║
╠═══════════════════════════════╬═════════════════════════════════════════╣
║ ExpirationNotification        ║ Notifies via EventBridge N days before  ║
║                               ║ the expiration date                     ║
╠═══════════════════════════════╬═════════════════════════════════════════╣
║ NoChangeNotification          ║ Notifies via EventBridge if the         ║
║                               ║ parameter was not modified in N days    ║
╚═══════════════════════════════╩═════════════════════════════════════════╝
```

```json
// Example of combined policy in JSON (passed in --policies)
[
  {
    "Type": "Expiration",
    "Version": "1.0",
    "Attributes": {
      "Timestamp": "2026-12-31T23:59:59.000Z"
    }
  },
  {
    "Type": "ExpirationNotification",
    "Version": "1.0",
    "Attributes": {
      "Before": "15",
      "Unit": "Days"
    }
  },
  {
    "Type": "NoChangeNotification",
    "Version": "1.0",
    "Attributes": {
      "After": "30",
      "Unit": "Days"
    }
  }
]
```

[FACT] Policy events are emitted in EventBridge with source `aws.ssm` and detail-type `Parameter Store Policy Action`.

---

## 5. CDK Python — Complete Hierarchy

```python
from aws_cdk import (
    Stack, SecretValue, aws_ssm as ssm,
    aws_kms as kms, aws_iam as iam,
    aws_lambda as _lambda, aws_ec2 as ec2,
)
from constructs import Construct

class ParameterStoreStack(Stack):
    def __init__(self, scope: Construct, construct_id: str,
                 app_name: str = "checkout", env: str = "prod",
                 **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        prefix = f"/{app_name}/{env}"

        # ──────────────────────────────────────────────────────────────
        # CMK for SecureString (granular control)
        # ──────────────────────────────────────────────────────────────
        param_key = kms.Key(self, "ParamKey",
            description=f"CMK for SSM Parameter Store — {app_name}/{env}",
            enable_key_rotation=True,
            alias=f"alias/ssm-{app_name}-{env}",
        )

        # ──────────────────────────────────────────────────────────────
        # String parameters — non-sensitive configurations
        # ──────────────────────────────────────────────────────────────
        ssm.StringParameter(self, "DBHost",
            parameter_name=f"{prefix}/database/host",
            string_value="my-cluster.cluster-xxxxx.us-east-1.rds.amazonaws.com",
            description="RDS PostgreSQL cluster endpoint",
            tier=ssm.ParameterTier.STANDARD,
        )

        ssm.StringParameter(self, "DBPort",
            parameter_name=f"{prefix}/database/port",
            string_value="5432",
            tier=ssm.ParameterTier.STANDARD,
        )

        ssm.StringParameter(self, "DBName",
            parameter_name=f"{prefix}/database/name",
            string_value="appdb",
            tier=ssm.ParameterTier.STANDARD,
        )

        ssm.StringParameter(self, "RedisEndpoint",
            parameter_name=f"{prefix}/redis/endpoint",
            string_value="my-redis.xxxxx.ng.0001.use1.cache.amazonaws.com:6379",
            tier=ssm.ParameterTier.STANDARD,
        )

        # ──────────────────────────────────────────────────────────────
        # StringList — list of allowed security groups
        # ──────────────────────────────────────────────────────────────
        ssm.StringListParameter(self, "AllowedSGs",
            parameter_name=f"{prefix}/network/allowed-security-groups",
            string_list_value=["sg-aaa111", "sg-bbb222", "sg-ccc333"],
            description="Security groups allowed for the load balancer",
        )

        # ──────────────────────────────────────────────────────────────
        # SecureString — sensitive values (API keys, tokens)
        # Note: CDK L2 ssm.StringParameter does not support SecureString directly.
        # Use CfnParameter (L1) for SecureString with CMK.
        # ──────────────────────────────────────────────────────────────
        ssm.CfnParameter(self, "DatadogAPIKey",
            name=f"{prefix}/observability/datadog-api-key",
            type="SecureString",
            value="PLACEHOLDER_ROTATED_EXTERNALLY",   # will be overwritten via CLI/pipeline
            description="Datadog API Key — SecureString with CMK",
            key_id=param_key.key_id,
            tier="Standard",
        )

        ssm.CfnParameter(self, "JWTSigningKey",
            name=f"{prefix}/auth/jwt-signing-key",
            type="SecureString",
            value="PLACEHOLDER",
            description="JWT signing key — SecureString with CMK",
            key_id=param_key.key_id,
            tier="Advanced",            # Advanced to enable Parameter Policies
            policies="""[
                {"Type":"NoChangeNotification","Version":"1.0","Attributes":{"After":"30","Unit":"Days"}},
                {"Type":"ExpirationNotification","Version":"1.0","Attributes":{"Before":"7","Unit":"Days"}}
            ]""",
        )

        # ──────────────────────────────────────────────────────────────
        # Lambda — reads full hierarchy at initialization
        # ──────────────────────────────────────────────────────────────
        app_fn = _lambda.Function(self, "AppFunction",
            runtime=_lambda.Runtime.PYTHON_3_12,
            handler="index.handler",
            code=_lambda.Code.from_inline("""
import boto3, os, json

ssm = boto3.client('ssm')
_config_cache = None

def load_config():
    global _config_cache
    if _config_cache:
        return _config_cache
    prefix = os.environ['PARAM_PREFIX']
    config = {}
    paginator = ssm.get_paginator('get_parameters_by_path')
    for page in paginator.paginate(
        Path=prefix,
        Recursive=True,
        WithDecryption=True,   # decrypts SecureString automatically
    ):
        for param in page['Parameters']:
            # Extract key relative to prefix: /checkout/prod/database/host → database/host
            key = param['Name'].replace(prefix + '/', '').replace('/', '_')
            config[key] = param['Value']
    _config_cache = config
    return config

def handler(event, context):
    config = load_config()
    return {'statusCode': 200, 'body': json.dumps({'dbHost': config.get('database_host')})}
"""),
            environment={
                "PARAM_PREFIX": f"{prefix}",
            },
        )

        # Minimum permissions — read-only on this application's prefix
        app_fn.add_to_role_policy(iam.PolicyStatement(
            actions=["ssm:GetParameter", "ssm:GetParametersByPath", "ssm:GetParameters"],
            resources=[
                f"arn:aws:ssm:{self.region}:{self.account}:parameter{prefix}/*"
            ],
        ))
        # KMS permission to decrypt SecureStrings with CMK
        param_key.grant_decrypt(app_fn)


class ParameterReferenceStack(Stack):
    """
    Demonstrates how to reference SSM parameters in other CDK resources.
    """
    def __init__(self, scope: Construct, construct_id: str, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # Reference to already existing parameter (does not create new one)
        db_host_param = ssm.StringParameter.from_string_parameter_name(
            self, "DBHostRef",
            string_parameter_name="/checkout/prod/database/host",
        )

        # Read value at synth time (deploy-time resolution)
        db_host_value = ssm.StringParameter.value_for_string_parameter(
            self, "/checkout/prod/database/host"
        )

        # Specific version
        db_host_v2 = ssm.StringParameter.value_for_string_parameter(
            self, "/checkout/prod/database/host",
            # version=2  # specific version
        )
```

---

## 6. Python — Hierarchy Reading with Cache

```python
"""
Configuration loader based on SSM Parameter Store.
Pattern: loads full hierarchy on cold start, caches in memory.
"""
import os
import boto3
import logging
from typing import Any, Optional
from functools import lru_cache
from dataclasses import dataclass

logger = logging.getLogger(__name__)
ssm = boto3.client("ssm")


@dataclass
class AppConfig:
    """Typed application configuration."""
    db_host: str
    db_port: int
    db_name: str
    db_password: str         # SecureString — decrypted at runtime
    redis_endpoint: str
    jwt_signing_key: str     # SecureString
    feature_new_checkout: bool

    @classmethod
    def from_param_dict(cls, params: dict[str, str]) -> "AppConfig":
        return cls(
            db_host=params["database_host"],
            db_port=int(params.get("database_port", "5432")),
            db_name=params["database_name"],
            db_password=params["database_password"],
            redis_endpoint=params["redis_endpoint"],
            jwt_signing_key=params["auth_jwt-signing-key"],
            feature_new_checkout=params.get("feature-flags_new-checkout-enabled", "false").lower() == "true",
        )


def load_parameters_by_path(
    path_prefix: str,
    with_decryption: bool = True,
) -> dict[str, str]:
    """
    Loads all parameters under path_prefix recursively.
    Returns dict with key = relative path (/ replaced by _).
    E.g.: /checkout/prod/database/host → "database_host"
    """
    params: dict[str, str] = {}
    paginator = ssm.get_paginator("get_parameters_by_path")

    pages = paginator.paginate(
        Path=path_prefix,
        Recursive=True,
        WithDecryption=with_decryption,
        PaginationConfig={"PageSize": 10},  # maximum per page
    )

    for page in pages:
        for param in page.get("Parameters", []):
            relative_key = (
                param["Name"]
                .removeprefix(path_prefix)   # remove /checkout/prod
                .lstrip("/")                  # remove leading /
                .replace("/", "_")            # /database/host → database_host
            )
            params[relative_key] = param["Value"]
            logger.debug("Loaded param: %s (version %d)", param["Name"], param["Version"])

    logger.info("Loaded %d parameters from %s", len(params), path_prefix)
    return params


# Module-level cache — persists between Lambda invocations (warm start)
_config_cache: Optional[AppConfig] = None


def get_config(force_reload: bool = False) -> AppConfig:
    """
    Returns application configuration.
    Caches between Lambda invocations to avoid repeated SSM calls.
    """
    global _config_cache
    if _config_cache is None or force_reload:
        prefix = os.environ.get("PARAM_PREFIX", "/checkout/prod")
        params = load_parameters_by_path(prefix)
        _config_cache = AppConfig.from_param_dict(params)
    return _config_cache


def get_single_param(name: str, with_decryption: bool = True) -> str:
    """Fetches a single parameter by full name."""
    response = ssm.get_parameter(Name=name, WithDecryption=with_decryption)
    return response["Parameter"]["Value"]


def get_params_by_names(names: list[str], with_decryption: bool = True) -> dict[str, str]:
    """
    Fetches multiple parameters by name in a single call.
    GetParameters accepts up to 10 names per call.
    Reports invalid names in InvalidParameters.
    """
    result: dict[str, str] = {}

    # Process in batches of 10 (API limit)
    for i in range(0, len(names), 10):
        batch = names[i:i + 10]
        response = ssm.get_parameters(Names=batch, WithDecryption=with_decryption)

        for param in response.get("Parameters", []):
            result[param["Name"]] = param["Value"]

        invalid = response.get("InvalidParameters", [])
        if invalid:
            logger.warning("Invalid parameter names: %s", invalid)

    return result


# Lambda handler that uses configuration loaded from SSM
def lambda_handler(event: dict, context) -> dict:
    config = get_config()

    # Example: new feature flag accessible without reloading config
    # For feature flags, short TTL or periodic reloading is recommended
    if config.feature_new_checkout:
        return process_new_checkout(event, config)
    return process_legacy_checkout(event, config)


def process_new_checkout(event: dict, config: AppConfig) -> dict:
    return {"statusCode": 200, "flow": "new-checkout", "db": config.db_host}


def process_legacy_checkout(event: dict, config: AppConfig) -> dict:
    return {"statusCode": 200, "flow": "legacy", "db": config.db_host}
```

---

## 7. CLI — Essential Examples

```bash
# 1. Create String parameter in hierarchy
aws ssm put-parameter \
  --name "/checkout/prod/database/host" \
  --value "my-cluster.cluster-xxx.us-east-1.rds.amazonaws.com" \
  --type String \
  --description "RDS PostgreSQL prod endpoint" \
  --tier Standard \
  --tags Key=app,Value=checkout Key=env,Value=prod

# 2. Create SecureString with CMK
aws ssm put-parameter \
  --name "/checkout/prod/database/password" \
  --value "$(openssl rand -base64 32)" \
  --type SecureString \
  --key-id "alias/ssm-checkout-prod" \
  --tier Standard

# 3. Create Advanced parameter with Parameter Policies
aws ssm put-parameter \
  --name "/checkout/prod/auth/jwt-signing-key" \
  --value "$(openssl rand -base64 64)" \
  --type SecureString \
  --key-id "alias/ssm-checkout-prod" \
  --tier Advanced \
  --policies '[
    {"Type":"NoChangeNotification","Version":"1.0","Attributes":{"After":"30","Unit":"Days"}},
    {"Type":"ExpirationNotification","Version":"1.0","Attributes":{"Before":"7","Unit":"Days"}}
  ]'

# 4. GetParametersByPath — entire prod hierarchy (recursive)
aws ssm get-parameters-by-path \
  --path "/checkout/prod" \
  --recursive \
  --with-decryption \
  --query 'Parameters[*].{Name:Name,Value:Value,Type:Type,Version:Version}' \
  --output table

# 5. GetParametersByPath — only /database (non-recursive)
aws ssm get-parameters-by-path \
  --path "/checkout/prod/database" \
  --with-decryption \
  --query 'Parameters[*].{Name:Name,Value:Value}' \
  --output table

# 6. GetParametersByPath — with tier filter
aws ssm get-parameters-by-path \
  --path "/checkout/prod" \
  --recursive \
  --parameter-filters "Key=tier,Values=Advanced"

# 7. Fetch multiple parameters at once (max 10 per call)
aws ssm get-parameters \
  --names \
    "/checkout/prod/database/host" \
    "/checkout/prod/database/port" \
    "/checkout/prod/redis/endpoint" \
  --with-decryption \
  --query '{
    Parameters: Parameters[*].{Name:Name,Value:Value},
    Invalid: InvalidParameters
  }'

# 8. Version history of a parameter
aws ssm get-parameter-history \
  --name "/checkout/prod/database/password" \
  --with-decryption \
  --query 'Parameters[*].{Version:Version,Modified:LastModifiedDate,User:LastModifiedUser}' \
  --output table

# 9. Update value without overwriting (--overwrite omitted = error if exists)
#    With --overwrite = automatically increments version
aws ssm put-parameter \
  --name "/checkout/prod/feature-flags/new-checkout-enabled" \
  --value "true" \
  --type String \
  --overwrite

# 10. List parameters with name filter (prefix search)
aws ssm describe-parameters \
  --parameter-filters "Key=Name,Option=BeginsWith,Values=/checkout/prod/" \
  --query 'Parameters[*].{Name:Name,Type:Type,Tier:Tier,Modified:LastModifiedDate}' \
  --output table

# 11. Enable high-throughput (when needed, has cost)
aws ssm update-service-setting \
  --setting-id "arn:aws:ssm:us-east-1:$(aws sts get-caller-identity --query Account --output text):servicesetting/ssm/parameter-store/high-throughput-enabled" \
  --setting-value true

# 12. Set Intelligent-Tiering as default
aws ssm update-service-setting \
  --setting-id "arn:aws:ssm:us-east-1:$(aws sts get-caller-identity --query Account --output text):servicesetting/ssm/parameter-store/default-parameter-tier" \
  --setting-value Intelligent-Tiering

# 13. Cross-account: share Advanced parameter with another account
aws ssm add-tags-to-resource \
  --resource-type Parameter \
  --resource-id "/checkout/prod/shared-config" \
  --tags Key=shareable,Value=true
# (actual sharing uses Resource Share in RAM for Advanced parameters)
```

---

## 8. Dynamic References — CloudFormation / CDK

[FACT] CloudFormation and CDK support references to SSM parameters and Secrets Manager directly in resource properties via **dynamic references**, resolved at deploy-time.

```
Dynamic reference syntax:

  String/StringList:
    {{resolve:ssm:/checkout/prod/database/host}}
    {{resolve:ssm:/checkout/prod/database/host:2}}   ← specific version

  SecureString (only in properties that support it):
    {{resolve:ssm-secure:/checkout/prod/database/password}}
    {{resolve:ssm-secure:/checkout/prod/database/password:3}}

  Secrets Manager:
    {{resolve:secretsmanager:MySecret:SecretString:username}}
    {{resolve:secretsmanager:MySecret:SecretString:password::AWSPENDING}}
```

[FACT] Dynamic references `ssm-secure` (SecureString) have limited support — they only work in specific CloudFormation resource properties (e.g.: `AWS::RDS::DBInstance` MasterUserPassword). They **do not** work in arbitrary properties or in `Outputs`.

```python
# CDK Python — using StringParameter.value_for_string_parameter
# Resolves at deploy-time (CloudFormation token)
from aws_cdk import aws_ssm as ssm, aws_rds as rds

db_name_value = ssm.StringParameter.value_for_string_parameter(
    self, "/checkout/prod/database/name"
)

# StringParameter.value_from_lookup = resolves at SYNTH-time (useful for conditionals)
db_host_at_synth = ssm.StringParameter.value_from_lookup(
    self, "/checkout/prod/database/host"
)
# CAUTION: value_from_lookup requires that the parameter already exists in the account/region
# Returns dummy value during first synth (CDK bootstrap)
```

---

## 9. Decision: SSM Parameter Store vs Secrets Manager

[CONSENSUS] The central decision criterion is: **do you need native automatic rotation?** If yes, Secrets Manager. If not, SSM Parameter Store (Standard) is significantly cheaper.

```
╔══════════════════════════════════════╦═══════════════════╦═══════════════════════╗
║ Criterion                            ║ SSM Standard/Adv  ║ Secrets Manager       ║
╠══════════════════════════════════════╬═══════════════════╬═══════════════════════╣
║ Cost per secret/parameter            ║ Free (Std)        ║ $0.40/month           ║
║                                      ║ $0.05/month (Adv) ║                       ║
╠══════════════════════════════════════╬═══════════════════╬═══════════════════════╣
║ API call cost                        ║ Free (≤40 TPS)    ║ $0.05 / 10k calls     ║
║                                      ║ $0.05/10k (HT)    ║                       ║
╠══════════════════════════════════════╬═══════════════════╬═══════════════════════╣
║ Native automatic rotation            ║ No                ║ Yes (4h to 365 days)  ║
╠══════════════════════════════════════╬═══════════════════╬═══════════════════════╣
║ Staging labels (AWSCURRENT etc.)     ║ Simple versions   ║ Yes (complete)        ║
╠══════════════════════════════════════╬═══════════════════╬═══════════════════════╣
║ Default throughput                   ║ 40 TPS            ║ ~200 TPS (default)    ║
║ Maximum throughput                   ║ 10,000 TPS (paid) ║ ~10,000 TPS           ║
╠══════════════════════════════════════╬═══════════════════╬═══════════════════════╣
║ Maximum value size                   ║ 4 KB / 8 KB       ║ 64 KB                 ║
╠══════════════════════════════════════╬═══════════════════╬═══════════════════════╣
║ Native cross-account sharing         ║ Advanced only     ║ Yes (resource policy) ║
╠══════════════════════════════════════╬═══════════════════╬═══════════════════════╣
║ Parameter Policies (expiration etc.) ║ Advanced only     ║ N/A                   ║
╠══════════════════════════════════════╬═══════════════════╬═══════════════════════╣
║ CloudFormation/CDK integration       ║ Yes (all props)   ║ Partial (via dynamic) ║
╚══════════════════════════════════════╩═══════════════════╩═══════════════════════╝
```

### 9.1 Decision Tree

```
Do you need automatic rotation (RDS, Redis, API keys)?
  YES → Secrets Manager

Do you need to store more than 10,000 parameters per region?
  YES → SSM Advanced OR Secrets Manager

Is the value larger than 64 KB?
  YES → S3 (Parameter Store and Secrets Manager do not support it)

Do you need cross-account sharing without VPC Endpoint?
  YES → Secrets Manager (resource policy) OR SSM Advanced (RAM)

Do you have >1,000 parameters with intensive reads (>40 TPS)?
  YES, frequent → SSM with high-throughput enabled
  YES, but just config → cache in application, SSM Standard

Otherwise:
  Non-sensitive configurations   → SSM Standard (free, String)
  Simple sensitive configurations → SSM Standard SecureString (free)
  Credentials with rotation      → Secrets Manager
  Feature flags / configs        → SSM Standard (free, clean hierarchy)
```

### 9.2 Cost example for 1,000 parameters

```
SSM Parameter Store Standard:
  1,000 parameters × $0.00 = $0.00/month
  1,000,000 calls/month × $0.00 = $0.00/month (up to 40 TPS)
  TOTAL: $0.00/month

Secrets Manager:
  1,000 secrets × $0.40 = $400.00/month
  1,000,000 calls × $0.005 = $5.00/month
  TOTAL: ~$405.00/month

→ For simple configurations, SSM Standard is 100% cheaper.
→ For 10 database credentials with rotation, SM = $4.05/month — justified.
```

---

## 10. Pitfalls

[FACT] **Standard → Advanced is irreversible**: it is not possible to revert without deleting and recreating. If you delete and recreate, you lose version history and any CloudFormation reference that used the ARN.

[FACT] **`value_from_lookup` (synth-time) requires that the parameter exists**: if the parameter does not exist in the account/region during `cdk synth`, CDK returns a dummy value. The created resource will use the wrong value on the first deploy. Prefer `value_for_string_parameter` (deploy-time) when possible.

[FACT] **SecureString is not supported in all CloudFormation properties**: the dynamic reference `{{resolve:ssm-secure:...}}` only works in specific properties. For most properties, use Secrets Manager with `{{resolve:secretsmanager:...}}`.

[FACT] **GetParametersByPath has pagination of maximum 10 items per page**: if using `--max-results 10` (the maximum), large hierarchies need multiple pages. Always use paginator in the SDK or `--query` with NextToken in CLI.

[CONSENSUS] **Do not use SSM Parameter Store as a complete substitute for Secrets Manager for rotated credentials**: SSM has no native rotation mechanism. Implementing manual rotation via Lambda + EventBridge is possible, but reproduces the work that Secrets Manager already does natively.

[FACT] **Tags are not inherited by hierarchy**: tags applied to `/checkout/prod` are NOT automatically applied to `/checkout/prod/database/host`. Each parameter needs its own tags for chargeback.

[FACT] **SecureString with default key `aws/ssm` does not allow cross-account**: if an application in another account needs to read the parameter, use a CMK with a key policy that authorizes the external account.

---

## Reflection Exercise

A platform team needs to manage configurations for 5 applications, each in 3 environments (prod, staging, dev), totaling ≈ 2,000 parameters. Some configurations are sensitive (third-party API keys, tokens), others are not (endpoints, feature flags). There are also 8 database credentials that must be rotated monthly.

**Propose the complete configuration management architecture**, answering:

1. Which parameters go to SSM Standard, SSM Advanced, and Secrets Manager? Why?
2. How to structure the naming hierarchy so each application can read only its own parameters via IAM?
3. With 5 applications × 3 environments doing simultaneous cold starts (e.g.: deploy), how to avoid SSM throttling (40 TPS default)?
4. How do the 8 database credentials fit into this architecture alongside the other SSM parameters?
5. What is the estimated total monthly cost of the proposed solution?

---

## References

- [FACT] [AWS Systems Manager Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html) — docs.aws.amazon.com
- [FACT] [Managing parameter tiers](https://docs.aws.amazon.com/systems-manager/latest/userguide/parameter-store-advanced-parameters.html) — docs.aws.amazon.com
- [FACT] [Increasing or resetting Parameter Store throughput](https://docs.aws.amazon.com/systems-manager/latest/userguide/parameter-store-throughput.html) — docs.aws.amazon.com
- [FACT] [SecureString parameters](https://docs.aws.amazon.com/systems-manager/latest/userguide/sysman-paramstore-securestring.html) — docs.aws.amazon.com
- [FACT] [AWS KMS encryption for SSM Parameter Store SecureString](https://docs.aws.amazon.com/kms/latest/developerguide/services-parameter-store.html) — docs.aws.amazon.com
