# Sessão 045 — SSM Parameter Store: Hierarquias, SecureString e Decisão vs Secrets Manager

**Duração estimada:** 60 min  
**Pré-requisito:** session-044 (Secrets Manager, rotação, staging labels)

---

## Objetivos da sessão

- Criar e recuperar parâmetros em hierarquia com `get-parameters-by-path` (recursivo, filtros)
- Entender os três tipos: String, StringList, SecureString (com chave padrão vs CMK)
- Comparar tiers Standard vs Advanced vs Intelligent-Tiering (custo, limites, políticas)
- Usar Parameter Policies (Expiration, ExpirationNotification, NoChangeNotification) — Advanced only
- Referenciar parâmetros em CDK/CloudFormation via dynamic references
- Documentar critério de decisão entre SSM Parameter Store e Secrets Manager

---

## 1. Tipos de Parâmetro

[FATO] Parameter Store suporta três tipos:

```
╔═════════════════╦══════════════════════════════════════════════════════╗
║ Tipo            ║ Características                                      ║
╠═════════════════╬══════════════════════════════════════════════════════╣
║ String          ║ Plaintext. Qualquer texto. Ex.: ARN, endpoint, flag  ║
╠═════════════════╬══════════════════════════════════════════════════════╣
║ StringList      ║ Lista de strings separadas por vírgula.              ║
║                 ║ Ex.: "sg-abc,sg-def" (usada com aws:ssm:type)        ║
╠═════════════════╬══════════════════════════════════════════════════════╣
║ SecureString    ║ Criptografado com KMS em repouso e em trânsito.      ║
║                 ║ Chave padrão aws/ssm ou CMK do cliente.              ║
╚═════════════════╩══════════════════════════════════════════════════════╝
```

### 1.1 SecureString — Chave Padrão vs CMK

[FATO] SecureString com chave padrão `aws/ssm`:
- Gratuita (sem cobranças KMS adicionais)
- Chave gerenciada pela AWS — sem controle de key policy
- Não permite cross-account decrypt
- Uma chave por região por conta — sem isolamento por parâmetro

[FATO] SecureString com CMK (Customer Managed Key):
- Cobrança KMS por operação de criptografia
- Controle granular via key policy (quem pode encriptar/decriptar)
- Permite cross-account: conta B pode decriptar se tiver permissão na key policy da conta A
- Permite audit trail via CloudTrail por chave

[FATO] O throughput de SecureString é limitado pelo throughput do KMS da região (5.500–10.000 TPS dependendo de região e tipo de chave).

---

## 2. Tiers: Standard, Advanced, Intelligent-Tiering

[FATO] Tabela comparativa completa (documentação AWS):

```
╔═══════════════════════════════╦═══════════════╦══════════════╗
║ Característica                ║ Standard      ║ Advanced     ║
╠═══════════════════════════════╬═══════════════╬══════════════╣
║ Parâmetros por região/conta   ║ 10.000        ║ 100.000      ║
║ Tamanho máximo do valor       ║ 4 KB          ║ 8 KB         ║
║ Parameter Policies            ║ Não           ║ Sim          ║
║ Compartilhamento cross-account║ Não           ║ Sim          ║
║ Custo de armazenamento        ║ Grátis        ║ $0,05/param/mês ║
╚═══════════════════════════════╩═══════════════╩══════════════╝
```

[FATO] Promoção Standard → Advanced é **irreversível**: não é possível reverter um parâmetro Advanced para Standard sem deletar e recriar (o que causaria perda de dados e histórico de versões). Se quiser economizar, deve excluir e recriar como Standard.

[FATO] **Intelligent-Tiering**: Parameter Store avalia cada `PutParameter` e cria Standard automaticamente, exceto quando: valor > 4 KB, conta já tem ≥ 10.000 parâmetros, ou parâmetro usa policies. Nesse caso, promove para Advanced. Recomendado para times com crescimento incerto de uso.

### 2.1 Throughput

[FATO] O throughput padrão do Parameter Store é de **40 TPS**. Com high-throughput habilitado (pago), aumenta para até **10.000 TPS**.

[FATO] O custo do high-throughput é $0,05 por 10.000 chamadas de API (quando habilitado). Se você usar apenas 40 TPS ou menos, não há razão para habilitar high-throughput.

---

## 3. Hierarquias de Parâmetros

[FATO] Parameter Store suporta estrutura hierárquica por path, com até **15 níveis** de profundidade separados por `/`. O nome completo (path) tem limite de 2.048 caracteres.

### 3.1 Convenção de hierarquia recomendada

```
/aplicação/ambiente/componente/chave

Exemplos práticos:
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

### 3.2 GetParametersByPath — Recuperar toda uma hierarquia

[FATO] `GetParametersByPath` retorna parâmetros sob um path prefix. Com `--recursive`, inclui todos os subníveis. Com `--with-decryption`, decripta SecureString automaticamente.

```
/checkout/prod/database/host     ←─┐
/checkout/prod/database/port     ←─┤ GetParametersByPath("/checkout/prod/database", recursive=False)
/checkout/prod/database/password ←─┘

/checkout/prod/database/host     ←─┐
/checkout/prod/database/port     ←─┤ GetParametersByPath("/checkout/prod", recursive=True)
/checkout/prod/database/password ←─┤
/checkout/prod/redis/endpoint    ←─┘
```

### 3.3 IAM — Escopo por hierarquia

[FATO] Permissões IAM podem ser escopadas por path prefix usando o ARN do parâmetro:

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

[FATO] Parameter Policies permitem associar comportamentos automáticos a um parâmetro Advanced. Três tipos:

```
╔═══════════════════════════════╦═════════════════════════════════════════╗
║ Tipo de Policy                ║ Comportamento                           ║
╠═══════════════════════════════╬═════════════════════════════════════════╣
║ Expiration                    ║ Deleta automaticamente o parâmetro em   ║
║                               ║ uma data/hora específica                ║
╠═══════════════════════════════╬═════════════════════════════════════════╣
║ ExpirationNotification        ║ Notifica via EventBridge N dias antes    ║
║                               ║ da data de expiração                    ║
╠═══════════════════════════════╬═════════════════════════════════════════╣
║ NoChangeNotification          ║ Notifica via EventBridge se o parâmetro ║
║                               ║ não foi modificado em N dias            ║
╚═══════════════════════════════╩═══════════════════════════════════════╝
```

```json
// Exemplo de policy combinada em JSON (passada no --policies)
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

[FATO] Eventos de Policy são emitidos no EventBridge com source `aws.ssm` e detail-type `Parameter Store Policy Action`.

---

## 5. CDK Python — Hierarquia completa

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
        # CMK para SecureString (controle granular)
        # ──────────────────────────────────────────────────────────────
        param_key = kms.Key(self, "ParamKey",
            description=f"CMK para SSM Parameter Store — {app_name}/{env}",
            enable_key_rotation=True,
            alias=f"alias/ssm-{app_name}-{env}",
        )

        # ──────────────────────────────────────────────────────────────
        # Parâmetros String — configurações não-sensíveis
        # ──────────────────────────────────────────────────────────────
        ssm.StringParameter(self, "DBHost",
            parameter_name=f"{prefix}/database/host",
            string_value="my-cluster.cluster-xxxxx.us-east-1.rds.amazonaws.com",
            description="Endpoint do cluster RDS PostgreSQL",
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
        # StringList — lista de security groups permitidos
        # ──────────────────────────────────────────────────────────────
        ssm.StringListParameter(self, "AllowedSGs",
            parameter_name=f"{prefix}/network/allowed-security-groups",
            string_list_value=["sg-aaa111", "sg-bbb222", "sg-ccc333"],
            description="Security groups permitidos para o balanceador",
        )

        # ──────────────────────────────────────────────────────────────
        # SecureString — valores sensíveis (API keys, tokens)
        # Nota: CDK L2 ssm.StringParameter não suporta SecureString diretamente.
        # Use CfnParameter (L1) para SecureString com CMK.
        # ──────────────────────────────────────────────────────────────
        ssm.CfnParameter(self, "DatadogAPIKey",
            name=f"{prefix}/observability/datadog-api-key",
            type="SecureString",
            value="PLACEHOLDER_ROTATED_EXTERNALLY",   # será sobrescrito via CLI/pipeline
            description="Datadog API Key — SecureString com CMK",
            key_id=param_key.key_id,
            tier="Standard",
        )

        ssm.CfnParameter(self, "JWTSigningKey",
            name=f"{prefix}/auth/jwt-signing-key",
            type="SecureString",
            value="PLACEHOLDER",
            description="JWT signing key — SecureString com CMK",
            key_id=param_key.key_id,
            tier="Advanced",            # Advanced para poder usar Parameter Policies
            policies="""[
                {"Type":"NoChangeNotification","Version":"1.0","Attributes":{"After":"30","Unit":"Days"}},
                {"Type":"ExpirationNotification","Version":"1.0","Attributes":{"Before":"7","Unit":"Days"}}
            ]""",
        )

        # ──────────────────────────────────────────────────────────────
        # Lambda — leitura da hierarquia completa na inicialização
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
        WithDecryption=True,   # decripta SecureString automaticamente
    ):
        for param in page['Parameters']:
            # Extrai chave relativa ao prefix: /checkout/prod/database/host → database/host
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

        # Permissões mínimas — apenas leitura no prefix desta aplicação
        app_fn.add_to_role_policy(iam.PolicyStatement(
            actions=["ssm:GetParameter", "ssm:GetParametersByPath", "ssm:GetParameters"],
            resources=[
                f"arn:aws:ssm:{self.region}:{self.account}:parameter{prefix}/*"
            ],
        ))
        # Permissão KMS para decriptar SecureStrings com CMK
        param_key.grant_decrypt(app_fn)


class ParameterReferenceStack(Stack):
    """
    Demonstra como referenciar parâmetros SSM em outros recursos CDK.
    """
    def __init__(self, scope: Construct, construct_id: str, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # Referência a parâmetro já existente (não cria novo)
        db_host_param = ssm.StringParameter.from_string_parameter_name(
            self, "DBHostRef",
            string_parameter_name="/checkout/prod/database/host",
        )

        # Leitura do valor em tempo de síntese (deploy-time resolution)
        db_host_value = ssm.StringParameter.value_for_string_parameter(
            self, "/checkout/prod/database/host"
        )

        # Versão específica
        db_host_v2 = ssm.StringParameter.value_for_string_parameter(
            self, "/checkout/prod/database/host",
            # version=2  # versão específica
        )
```

---

## 6. Python — Leitura de hierarquia com cache

```python
"""
Carregador de configuração baseado em SSM Parameter Store.
Padrão: carrega hierarquia completa no cold start, cacheia em memória.
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
    """Configuração tipada da aplicação."""
    db_host: str
    db_port: int
    db_name: str
    db_password: str         # SecureString — decriptado em runtime
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
    Carrega todos os parâmetros sob path_prefix recursivamente.
    Retorna dict com chave = path relativo (/ substituído por _).
    Ex.: /checkout/prod/database/host → "database_host"
    """
    params: dict[str, str] = {}
    paginator = ssm.get_paginator("get_parameters_by_path")

    pages = paginator.paginate(
        Path=path_prefix,
        Recursive=True,
        WithDecryption=with_decryption,
        PaginationConfig={"PageSize": 10},  # máximo por página
    )

    for page in pages:
        for param in page.get("Parameters", []):
            relative_key = (
                param["Name"]
                .removeprefix(path_prefix)   # remove /checkout/prod
                .lstrip("/")                  # remove / inicial
                .replace("/", "_")            # /database/host → database_host
            )
            params[relative_key] = param["Value"]
            logger.debug("Loaded param: %s (version %d)", param["Name"], param["Version"])

    logger.info("Loaded %d parameters from %s", len(params), path_prefix)
    return params


# Cache em nível de módulo — persiste entre invocações Lambda (warm start)
_config_cache: Optional[AppConfig] = None


def get_config(force_reload: bool = False) -> AppConfig:
    """
    Retorna configuração da aplicação.
    Cacheia entre invocações Lambda para evitar chamadas SSM repetidas.
    """
    global _config_cache
    if _config_cache is None or force_reload:
        prefix = os.environ.get("PARAM_PREFIX", "/checkout/prod")
        params = load_parameters_by_path(prefix)
        _config_cache = AppConfig.from_param_dict(params)
    return _config_cache


def get_single_param(name: str, with_decryption: bool = True) -> str:
    """Busca um único parâmetro pelo nome completo."""
    response = ssm.get_parameter(Name=name, WithDecryption=with_decryption)
    return response["Parameter"]["Value"]


def get_params_by_names(names: list[str], with_decryption: bool = True) -> dict[str, str]:
    """
    Busca múltiplos parâmetros por nome em uma única chamada.
    GetParameters aceita até 10 nomes por chamada.
    Invalida nomes inválidos reportados em InvalidParameters.
    """
    result: dict[str, str] = {}

    # Processar em lotes de 10 (limite da API)
    for i in range(0, len(names), 10):
        batch = names[i:i + 10]
        response = ssm.get_parameters(Names=batch, WithDecryption=with_decryption)

        for param in response.get("Parameters", []):
            result[param["Name"]] = param["Value"]

        invalid = response.get("InvalidParameters", [])
        if invalid:
            logger.warning("Invalid parameter names: %s", invalid)

    return result


# Lambda handler que usa configuração carregada do SSM
def lambda_handler(event: dict, context) -> dict:
    config = get_config()

    # Exemplo: nova feature flag acessível sem recarregar configuração
    # Para feature flags, recomenda-se TTL curto ou recarregar periodicamente
    if config.feature_new_checkout:
        return process_new_checkout(event, config)
    return process_legacy_checkout(event, config)


def process_new_checkout(event: dict, config: AppConfig) -> dict:
    return {"statusCode": 200, "flow": "new-checkout", "db": config.db_host}


def process_legacy_checkout(event: dict, config: AppConfig) -> dict:
    return {"statusCode": 200, "flow": "legacy", "db": config.db_host}
```

---

## 7. CLI — Exemplos Essenciais

```bash
# 1. Criar parâmetro String na hierarquia
aws ssm put-parameter \
  --name "/checkout/prod/database/host" \
  --value "my-cluster.cluster-xxx.us-east-1.rds.amazonaws.com" \
  --type String \
  --description "Endpoint RDS PostgreSQL prod" \
  --tier Standard \
  --tags Key=app,Value=checkout Key=env,Value=prod

# 2. Criar SecureString com CMK
aws ssm put-parameter \
  --name "/checkout/prod/database/password" \
  --value "$(openssl rand -base64 32)" \
  --type SecureString \
  --key-id "alias/ssm-checkout-prod" \
  --tier Standard

# 3. Criar parâmetro Advanced com Parameter Policies
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

# 4. GetParametersByPath — toda a hierarquia de prod (recursivo)
aws ssm get-parameters-by-path \
  --path "/checkout/prod" \
  --recursive \
  --with-decryption \
  --query 'Parameters[*].{Name:Name,Value:Value,Type:Type,Version:Version}' \
  --output table

# 5. GetParametersByPath — apenas /database (sem recursão)
aws ssm get-parameters-by-path \
  --path "/checkout/prod/database" \
  --with-decryption \
  --query 'Parameters[*].{Name:Name,Value:Value}' \
  --output table

# 6. GetParametersByPath — com filtro por tier
aws ssm get-parameters-by-path \
  --path "/checkout/prod" \
  --recursive \
  --parameter-filters "Key=tier,Values=Advanced"

# 7. Buscar múltiplos parâmetros de uma vez (máx. 10 por chamada)
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

# 8. Histórico de versões de um parâmetro
aws ssm get-parameter-history \
  --name "/checkout/prod/database/password" \
  --with-decryption \
  --query 'Parameters[*].{Version:Version,Modified:LastModifiedDate,User:LastModifiedUser}' \
  --output table

# 9. Atualizar valor sem sobrescrever (--overwrite omitido = erro se existir)
#    Com --overwrite = incrementa versão automaticamente
aws ssm put-parameter \
  --name "/checkout/prod/feature-flags/new-checkout-enabled" \
  --value "true" \
  --type String \
  --overwrite

# 10. Listar parâmetros com filtro por nome (busca prefix)
aws ssm describe-parameters \
  --parameter-filters "Key=Name,Option=BeginsWith,Values=/checkout/prod/" \
  --query 'Parameters[*].{Name:Name,Type:Type,Tier:Tier,Modified:LastModifiedDate}' \
  --output table

# 11. Habilitar high-throughput (quando necessário, tem custo)
aws ssm update-service-setting \
  --setting-id "arn:aws:ssm:us-east-1:$(aws sts get-caller-identity --query Account --output text):servicesetting/ssm/parameter-store/high-throughput-enabled" \
  --setting-value true

# 12. Configurar Intelligent-Tiering como padrão
aws ssm update-service-setting \
  --setting-id "arn:aws:ssm:us-east-1:$(aws sts get-caller-identity --query Account --output text):servicesetting/ssm/parameter-store/default-parameter-tier" \
  --setting-value Intelligent-Tiering

# 13. Cross-account: compartilhar parâmetro Advanced com outra conta
aws ssm add-tags-to-resource \
  --resource-type Parameter \
  --resource-id "/checkout/prod/shared-config" \
  --tags Key=shareable,Value=true
# (o compartilhamento real usa Resource Share no RAM para parâmetros Advanced)
```

---

## 8. Dynamic References — CloudFormation / CDK

[FATO] CloudFormation e CDK suportam referências a parâmetros SSM e Secrets Manager diretamente em propriedades de recursos via **dynamic references**, resolvidas em deploy-time.

```
Sintaxe das dynamic references:

  String/StringList:
    {{resolve:ssm:/checkout/prod/database/host}}
    {{resolve:ssm:/checkout/prod/database/host:2}}   ← versão específica

  SecureString (somente em propriedades que suportam):
    {{resolve:ssm-secure:/checkout/prod/database/password}}
    {{resolve:ssm-secure:/checkout/prod/database/password:3}}

  Secrets Manager:
    {{resolve:secretsmanager:MySecret:SecretString:username}}
    {{resolve:secretsmanager:MySecret:SecretString:password::AWSPENDING}}
```

[FATO] Dynamic references `ssm-secure` (SecureString) têm suporte limitado — só funcionam em propriedades específicas de recursos CloudFormation (ex.: `AWS::RDS::DBInstance` MasterUserPassword). **Não** funcionam em propriedades arbitrárias ou em `Outputs`.

```python
# CDK Python — usando StringParameter.value_for_string_parameter
# Resolve em deploy-time (CloudFormation token)
from aws_cdk import aws_ssm as ssm, aws_rds as rds

db_name_value = ssm.StringParameter.value_for_string_parameter(
    self, "/checkout/prod/database/name"
)

# StringParameter.value_from_lookup = resolve em SYNTH-time (útil para conditionals)
db_host_at_synth = ssm.StringParameter.value_from_lookup(
    self, "/checkout/prod/database/host"
)
# CUIDADO: value_from_lookup requer que o parâmetro já exista na conta/região
# Retorna dummy value durante primeiro synth (CDK bootstrap)
```

---

## 9. Decisão: SSM Parameter Store vs Secrets Manager

[CONSENSO] O critério de decisão central é: **você precisa de rotação automática nativa?** Se sim, Secrets Manager. Se não, SSM Parameter Store (Standard) é significativamente mais barato.

```
╔══════════════════════════════════════╦═══════════════════╦═══════════════════════╗
║ Critério                             ║ SSM Standard/Adv  ║ Secrets Manager       ║
╠══════════════════════════════════════╬═══════════════════╬═══════════════════════╣
║ Custo por segredo/parâmetro          ║ Grátis (Std)      ║ $0,40/mês             ║
║                                      ║ $0,05/mês (Adv)   ║                       ║
╠══════════════════════════════════════╬═══════════════════╬═══════════════════════╣
║ Custo de API call                    ║ Grátis (≤40 TPS)  ║ $0,05 / 10k calls     ║
║                                      ║ $0,05/10k (HT)    ║                       ║
╠══════════════════════════════════════╬═══════════════════╬═══════════════════════╣
║ Rotação automática nativa            ║ Não               ║ Sim (4h a 365 dias)   ║
╠══════════════════════════════════════╬═══════════════════╬═══════════════════════╣
║ Staging labels (AWSCURRENT etc.)     ║ Versões simples   ║ Sim (completo)        ║
╠══════════════════════════════════════╬═══════════════════╬═══════════════════════╣
║ Throughput padrão                    ║ 40 TPS            ║ ~200 TPS (default)    ║
║ Throughput máximo                    ║ 10.000 TPS (pago) ║ ~10.000 TPS           ║
╠══════════════════════════════════════╬═══════════════════╬═══════════════════════╣
║ Tamanho máximo do valor              ║ 4 KB / 8 KB       ║ 64 KB                 ║
╠══════════════════════════════════════╬═══════════════════╬═══════════════════════╣
║ Cross-account sharing nativo         ║ Advanced apenas   ║ Sim (resource policy) ║
╠══════════════════════════════════════╬═══════════════════╬═══════════════════════╣
║ Parameter Policies (expiração etc.)  ║ Advanced apenas   ║ N/A                   ║
╠══════════════════════════════════════╬═══════════════════╬═══════════════════════╣
║ Integração CloudFormation/CDK        ║ Sim (todas props) ║ Parcial (via dynamic) ║
╚══════════════════════════════════════╩═══════════════════╩═══════════════════════╝
```

### 9.1 Árvore de Decisão

```
Você precisa de rotação automática (RDS, Redis, API keys)?
  SIM → Secrets Manager

Você precisa armazenar mais de 10.000 parâmetros por região?
  SIM → SSM Advanced OU Secrets Manager

O valor tem mais de 64 KB?
  SIM → S3 (Parameter Store e Secrets Manager não suportam)

Você precisa compartilhar cross-account sem VPC Endpoint?
  SIM → Secrets Manager (resource policy) OU SSM Advanced (RAM)

Você tem >1.000 parâmetros com leitura intensa (>40 TPS)?
  SIM, frequente → SSM com high-throughput habilitado
  SIM, mas é só config → cache na aplicação, SSM Standard

Caso contrário:
  Configurações não-sensíveis    → SSM Standard (grátis, String)
  Configurações sensíveis simples → SSM Standard SecureString (grátis)
  Credenciais com rotação         → Secrets Manager
  Feature flags / configs         → SSM Standard (grátis, hierarquia limpa)
```

### 9.2 Exemplo de custo para 1.000 parâmetros

```
SSM Parameter Store Standard:
  1.000 parâmetros × $0,00 = $0,00/mês
  1.000.000 chamadas/mês × $0,00 = $0,00/mês (até 40 TPS)
  TOTAL: $0,00/mês

Secrets Manager:
  1.000 segredos × $0,40 = $400,00/mês
  1.000.000 chamadas × $0,005 = $5,00/mês
  TOTAL: ~$405,00/mês

→ Para configurações simples, SSM Standard é 100% mais barato.
→ Para 10 credenciais de banco com rotação, SM = $4,05/mês — justificado.
```

---

## 10. Armadilhas (Pitfalls)

[FATO] **Standard → Advanced é irreversível**: não é possível reverter sem deletar e recriar. Se deletar e recriar, perde o histórico de versões e qualquer referência CloudFormation que usava o ARN.

[FATO] **`value_from_lookup` (synth-time) requer que o parâmetro exista**: se o parâmetro não existir na conta/região durante `cdk synth`, o CDK retorna um dummy value. O recurso criado vai usar o valor errado no primeiro deploy. Prefira `value_for_string_parameter` (deploy-time) quando possível.

[FATO] **SecureString não é suportado em todas as propriedades CloudFormation**: a dynamic reference `{{resolve:ssm-secure:...}}` só funciona em propriedades específicas. Para maioria das propriedades, usar Secrets Manager com `{{resolve:secretsmanager:...}}`.

[FATO] **GetParametersByPath tem paginação de máximo 10 itens por página**: se usar `--max-results 10` (o máximo), hierarquias grandes precisam de múltiplas páginas. Sempre use paginator no SDK ou `--query` com NextToken no CLI.

[CONSENSO] **Não usar SSM Parameter Store como substituto completo do Secrets Manager para credenciais rotacionadas**: SSM não tem mecanismo nativo de rotação. Implementar rotação manual via Lambda + EventBridge é possível, mas reproduz o trabalho que Secrets Manager já faz nativamente.

[FATO] **Tags não são herdadas por hierarquia**: tags aplicadas a `/checkout/prod` NÃO são automaticamente aplicadas a `/checkout/prod/database/host`. Cada parâmetro precisa de suas próprias tags para chargeback.

[FATO] **SecureString com chave padrão `aws/ssm` não permite cross-account**: se uma aplicação em outra conta precisa ler o parâmetro, use CMK com key policy que autorize a conta externa.

---

## Exercício de Reflexão

Um time de plataforma precisa gerenciar configurações para 5 aplicações, cada uma em 3 ambientes (prod, staging, dev), totalizando ≈ 2.000 parâmetros. Algumas configurações são sensíveis (API keys de terceiros, tokens), outras não (endpoints, feature flags). Há também 8 credenciais de banco de dados que devem ser rotacionadas mensalmente.

**Proponha a arquitetura completa de gerenciamento de configuração**, respondendo:

1. Quais parâmetros vão para SSM Standard, SSM Advanced e Secrets Manager? Por quê?
2. Como estruturar a hierarquia de nomes para que cada aplicação possa ler apenas seus próprios parâmetros via IAM?
3. Com 5 aplicações × 3 ambientes fazendo cold starts simultâneos (ex.: deploy), como evitar throttling do SSM (40 TPS padrão)?
4. Como as 8 credenciais de banco se encaixam nessa arquitetura junto com os demais parâmetros SSM?
5. Qual o custo mensal estimado total da solução proposta?

---

## Referências

- [FATO] [AWS Systems Manager Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html) — docs.aws.amazon.com
- [FATO] [Managing parameter tiers](https://docs.aws.amazon.com/systems-manager/latest/userguide/parameter-store-advanced-parameters.html) — docs.aws.amazon.com
- [FATO] [Increasing or resetting Parameter Store throughput](https://docs.aws.amazon.com/systems-manager/latest/userguide/parameter-store-throughput.html) — docs.aws.amazon.com
- [FATO] [SecureString parameters](https://docs.aws.amazon.com/systems-manager/latest/userguide/sysman-paramstore-securestring.html) — docs.aws.amazon.com
- [FATO] [AWS KMS encryption for SSM Parameter Store SecureString](https://docs.aws.amazon.com/kms/latest/developerguide/services-parameter-store.html) — docs.aws.amazon.com
