# Sessão session-027-alb-oidc-mtls-waf — ALB: OIDC nativo, mTLS e integração com WAF

**Duração estimada:** 60 minutos
**Pré-requisitos:** session-026-alb-listener-rules-weighted

---

## Objetivo

Ao final, você conseguirá configurar autenticação OIDC nativa no ALB (delegando auth para Cognito
ou qualquer IdP OIDC sem código no backend), habilitar mTLS com um trust store de CA, e associar
uma Web ACL do WAF a um ALB para filtrar requests antes de chegarem à aplicação.

---

## Contexto

[FATO] O ALB suporta três mecanismos de segurança no nível do load balancer — cada um atua em
uma camada diferente do modelo de ameaças e complementa os demais. OIDC nativo delega ao ALB a
função de autenticar usuários humanos via fluxo OAuth2/OIDC, eliminando a necessidade de
implementar esse fluxo em cada microserviço. mTLS vai além do TLS convencional e exige que o
*cliente* apresente um certificado X.509 assinado por uma CA confiável — útil para autenticação
machine-to-machine em fronteiras de serviço. O WAF (Web Application Firewall) atua antes mesmo de
o request chegar ao listener, filtrando tráfego malicioso com base em regras L7 (SQL injection,
XSS, rate limiting, geo-blocking).

[CONSENSO] A combinação dos três mecanismos segue o princípio de defesa em profundidade: o WAF
bloqueia ameaças conhecidas no perímetro, o mTLS garante que apenas clientes autorizados
estabelecem conexão TLS, e o OIDC garante que apenas usuários autenticados chegam ao backend.
Cada camada pode falhar ou ser contornada de formas diferentes — tê-las em conjunto reduz
significativamente a superfície de ataque.

---

## Conceitos principais

### 1. OIDC nativo no ALB — como funciona o fluxo de autenticação

[FATO] O ALB suporta dois tipos de ação de autenticação em regras de listener HTTPS:
`authenticate-oidc` (para qualquer IdP que implemente OIDC) e `authenticate-cognito` (atalho
para integração com Cognito User Pools). Ambos seguem o fluxo Authorization Code Grant do
OAuth 2.0. Somente listeners HTTPS suportam essas ações — em listeners HTTP a ação é recusada.

O fluxo de autenticação tem 11 passos documentados pela AWS:

```
Cliente                      ALB                         IdP (OIDC)          Backend
  |                           |                              |                   |
  |--- HTTPS request -------->|                              |                   |
  |                           |-- checa cookie AWSELB ------>|                   |
  |                           |   (não existe na 1ª vez)    |                   |
  |<-- 302 redirect ----------|                              |                   |
  |   (authorization endpoint)|                              |                   |
  |--- GET /authorize ---------------------------------------->|                |
  |                           |                              |-- user login UI  |
  |<-- auth code via redirect ----------------------------------------           |
  |--- POST /oauth2/idpresponse (auth code) -->|             |                   |
  |                           |-- POST /token ----------------->|               |
  |                           |<-- id_token + access_token -----|               |
  |                           |-- GET /userinfo (access_token) ->|              |
  |                           |<-- user claims (JSON) ----------|               |
  |<-- 302 redirect ----------|                              |                   |
  |   Set-Cookie: AWSELB=...  |                              |                   |
  |--- request original ------>|                             |                   |
  |                           |-- forward + X-AMZN-OIDC-* headers ------------->|
  |<-- response -------------------------------------------------- response -----|
```

[FATO] O cookie de sessão `AWSELB` é enviado ao cliente após autenticação bem-sucedida. Requisições
subsequentes apresentam esse cookie e o ALB valida localmente (sem roundtrip ao IdP) — saltando
direto para o passo 9. O cookie sempre carrega o atributo `Secure` (requer HTTPS). Para requests
CORS, inclui também `SameSite=None`.

[FATO] Se o total de user claims + access token exceder 11 KB, o ALB retorna HTTP 500 e incrementa
a métrica `ELBAuthUserClaimsSizeExceeded`. Cookies maiores que 4 KB são fragmentados em múltiplos
cookies (o ALB suporta até 4 shards, totalizando até 16 KB).

**Headers enviados ao backend:**

| Header | Conteúdo |
|--------|----------|
| `x-amzn-oidc-accesstoken` | Access token em plain text |
| `x-amzn-oidc-identity` | Claim `sub` do user info endpoint (plain text) |
| `x-amzn-oidc-data` | User claims em formato JWT (ES256, base64url encoded) |

[FATO] O campo `sub` é a forma canônica de identificar um usuário — o `sub` é estável e único por
IdP, enquanto campos como `email` podem mudar. O JWT em `x-amzn-oidc-data` é assinado pelo ALB com
ES256 (ECDSA P-256 + SHA256). A chave pública para verificação está disponível em:
`https://public-keys.auth.elb.<region>.amazonaws.com/<key-id>` (o `kid` está no header do JWT).

**Comportamento `OnUnauthenticatedRequest`:**

```
authenticate  → redireciona para IdP (padrão; bom para SPAs e apps web tradicionais)
allow         → passa o request sem claims (bom para apps com conteúdo público + privado)
deny          → retorna HTTP 401 (bom para APIs que fazem fetch em background — evita redirect loop)
```

**Session timeout:** padrão 7 dias, configurável de 1 segundo a 7 dias. Independente do TTL do
cookie (sempre 7 dias no atributo `Max-Age`) — o timeout real está criptografado dentro do valor
do cookie.

**Client login timeout:** fixo em 15 minutos. O usuário deve completar o login no IdP dentro
desse janela; caso contrário, recebe HTTP 401 e precisa recarregar a página.

### 2. Configuração OIDC — parâmetros obrigatórios e opcionais

[FATO] Para `authenticate-oidc`, os seguintes campos são obrigatórios na ação:

```
Issuer              — URL do issuer do IdP (ex: https://accounts.google.com)
AuthorizationEndpoint — URL de autorização (ex: https://accounts.google.com/o/oauth2/v2/auth)
TokenEndpoint       — URL de obtenção de tokens (ex: https://oauth2.googleapis.com/token)
UserInfoEndpoint    — URL de user info (ex: https://openidconnect.googleapis.com/v1/userinfo)
ClientId            — OAuth 2.0 client ID (registrado no IdP)
ClientSecret        — OAuth 2.0 client secret
```

[FATO] O DNS dos endpoints do IdP deve ser publicamente resolvível (mesmo que resolva para IPs
privados). O ALB se comunica com `TokenEndpoint` e `UserInfoEndpoint` — se o ALB for interno ou
usar `dualstack-without-public-ipv4`, é necessário um NAT Gateway para que o ALB alcance esses
endpoints. O ALB só suporta IPv4 nessa comunicação.

[FATO] A URL de callback que deve ser permitida no IdP segue o padrão:
`https://<DNS-do-ALB>/oauth2/idpresponse` ou `https://<CNAME>/oauth2/idpresponse`.

**Parâmetros opcionais relevantes:**

```
SessionCookieName         — nome do cookie (padrão: AWSELBAuthSessionCookie)
                            use nomes distintos para múltiplas regras com auth no mesmo listener
SessionTimeout            — tempo de sessão em segundos (padrão: 604800 = 7 dias)
Scope                     — scopes OAuth solicitados (padrão: "openid"; adicionar "email" para claims de email)
AuthenticationRequestExtraParams — parâmetros extras para o IdP (ex: {"prompt": "login"})
```

**Para Cognito (`authenticate-cognito`), os campos obrigatórios são:**
```
UserPoolArn        — ARN do User Pool
UserPoolClientId   — ID do app client (deve ter client secret + code grant habilitados)
UserPoolDomain     — domínio do User Pool (ex: minha-app.auth.us-east-1.amazoncognito.com)
```

### 3. mTLS no ALB — dois modos e trust store

[FATO] Mutual TLS (mTLS) inverte a autenticação TLS convencional: além do servidor apresentar seu
certificado ao cliente, o *cliente* deve apresentar um certificado X.509v3 assinado por uma CA que
o ALB reconhece como confiável. Isso é especialmente relevante em cenários B2B, machine-to-machine,
e zero-trust entre serviços.

O ALB oferece dois modos de mTLS:

```
┌─────────────────────────────────────────────────────────────────────┐
│                     mTLS PASSTHROUGH                                 │
│                                                                      │
│  Cliente → [cert chain] → ALB → [X-Amzn-Mtls-Clientcert header] → Backend │
│                                                                      │
│  O ALB NÃO verifica o certificado do cliente.                        │
│  Encaminha a cadeia completa (URL-encoded PEM) no header.            │
│  O backend é responsável por validar e autorizar.                    │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                      mTLS VERIFY                                     │
│                                                                      │
│  Cliente → [cert] → ALB → valida contra trust store → forward       │
│                           (rejeita se cert inválido/não confiável)   │
│                                                                      │
│  O ALB verifica a cadeia X.509 do cliente contra a trust store.      │
│  Envia metadados do cert ao backend via headers X-Amzn-Mtls-*.      │
└─────────────────────────────────────────────────────────────────────┘
```

[FATO] Para o modo Verify, é necessário criar um recurso **trust store** — uma coleção de
certificados raiz e intermediários de CAs confiáveis. O bundle de CAs é armazenado em S3 no
formato PEM e associado ao trust store. Para substituir o bundle, usa-se a API `ModifyTrustStore`
— não é possível adicionar certificados individuais, apenas substituir o bundle inteiro.

**Requisitos do arquivo de bundle PEM:**
- Formato PEM (Privacy Enhanced Mail)
- Cada certificado delimitado por `-----BEGIN CERTIFICATE-----` e `-----END CERTIFICATE-----`
- Comentários precedidos de `#` (sem traços `-` em comentários)
- Sem linhas em branco entre certificados
- Certificados suportados: X.509v3 com chaves RSA 2K–8K ou ECDSA (secp256r1/384r1/521r1)

**Headers enviados ao backend no modo Verify:**

```
X-Amzn-Mtls-Clientcert-Serial-Number  → número serial hexadecimal do leaf cert
X-Amzn-Mtls-Clientcert-Issuer         → DN do issuer (RFC 2253)
X-Amzn-Mtls-Clientcert-Subject        → DN do subject (RFC 2253)
X-Amzn-Mtls-Clientcert-Validity       → NotBefore=...;NotAfter=... (ISO 8601)
X-Amzn-Mtls-Clientcert-Leaf           → leaf cert em PEM URL-encoded
```

**Header no modo Passthrough:**
```
X-Amzn-Mtls-Clientcert  → cadeia completa em PEM URL-encoded (leaf → root)
```

[FATO] Session resumption (TLS session tickets) não é suportado nos modos mTLS passthrough
ou verify. Cada nova conexão TLS realiza o handshake completo.

**Advertise CA Subject Names:** quando habilitado, o ALB inclui a lista de DNs das CAs do trust
store na mensagem `CertificateRequest` do TLS handshake. O cliente pode usar essa lista para
selecionar o certificado correto — reduz erros de conexão quando o cliente tem múltiplos
certificados e não sabe qual usar.

### 4. AWS WAF — arquitetura e integração com ALB

[FATO] O AWS WAF v2 opera como um proxy L7 "inspetor" que avalia o request HTTP/HTTPS *antes*
de ele chegar ao ALB target. A unidade de configuração é a **Web ACL** (Access Control List),
que contém regras ordenadas por prioridade. O WAF avalia as regras em ordem crescente de
prioridade e aplica a ação da primeira regra que fizer match.

```
Internet
    │
    ▼
┌─────────────┐
│  AWS WAF    │  ← Web ACL (REGIONAL scope)
│  Web ACL    │     Regra 1 (prio 0): AWSManagedRulesCommonRuleSet
│             │     Regra 2 (prio 1): AWSManagedRulesSQLiRuleSet
│             │     Regra 3 (prio 2): custom rate-limit rule
│             │     Default action: Allow
└──────┬──────┘
       │  (só passa o que não foi bloqueado)
       ▼
┌─────────────┐
│     ALB     │  ← listener rules, OIDC, mTLS
└──────┬──────┘
       │
       ▼
   Targets (ECS, EC2, Lambda...)
```

[FATO] Para usar WAF com ALB, a Web ACL deve ser criada com escopo `REGIONAL` (não `CLOUDFRONT`).
Cada recurso AWS só pode ser associado a **uma** Web ACL de cada vez. O WAF e o ALB devem estar
na mesma região.

**Ações possíveis em cada regra WAF:**

```
Allow   → passa o request; termina avaliação (nenhuma outra regra é executada)
Block   → retorna HTTP 403 (padrão) ou resposta customizada; termina avaliação
Count   → incrementa contador; NÃO termina avaliação (continua para próximas regras)
CAPTCHA → apresenta desafio CAPTCHA ao usuário antes de prosseguir
Challenge → verifica se é browser real (JavaScript challenge silencioso)
```

[FATO] `Count` é especialmente útil para testar novas regras em produção sem bloquear tráfego
— você monitora as métricas de `CountedRequests` no CloudWatch antes de mudar para `Block`.

**AWS Managed Rule Groups relevantes para ALB:**

| Grupo | O que cobre |
|-------|-------------|
| `AWSManagedRulesCommonRuleSet` | OWASP Top 10 baseline (XSS, LFI, RFI, SSRF) |
| `AWSManagedRulesSQLiRuleSet` | SQL injection em query strings, body, headers |
| `AWSManagedRulesKnownBadInputsRuleSet` | Inputs malformados, Log4JRCE, Spring4Shell |
| `AWSManagedRulesAmazonIpReputationList` | IPs com reputação ruim (bots, scrapers, TOR) |
| `AWSManagedRulesAnonymousIpList` | Proxies anônimos, VPNs, TOR exit nodes |
| `AWSManagedRulesBotControlRuleSet` | Detecção de bots (pago por uso; cobra WCU extra) |

[FATO] Cada Web ACL tem uma cota de **WCU (WAF Capacity Units)**. O limite padrão é 1.500 WCU por
Web ACL. `AWSManagedRulesCommonRuleSet` consome 700 WCU, `AWSManagedRulesSQLiRuleSet` consome 200
WCU. Ao combinar múltiplos grupos, é necessário planejar o orçamento de WCUs.

[FATO] A associação WAF → ALB usa o ARN do ALB. Uma vez associada, o WAF inspeciona todos os
requests que chegam ao ALB — não é possível selecionar listeners específicos para inspecionar.

---

## Exemplo prático

**Cenário:** API de e-commerce com três requisitos de segurança:
1. Usuários web autenticam via Cognito (OIDC) antes de acessar `/dashboard/*`
2. Integração B2B (parceiros) autentica via certificado X.509 (mTLS verify) em `/api/b2b/*`
3. WAF protege todo o ALB contra OWASP Top 10 e SQL injection

### CDK Python — stack completa

```python
from aws_cdk import (
    Stack, Duration,
    aws_elasticloadbalancingv2 as elbv2,
    aws_elasticloadbalancingv2_actions as elbv2_actions,
    aws_cognito as cognito,
    aws_wafv2 as wafv2,
    aws_s3 as s3,
)
from constructs import Construct

class SecureAlbStack(Stack):
    def __init__(self, scope: Construct, id: str, **kwargs):
        super().__init__(scope, id, **kwargs)

        # ── 1. ALB (assumindo que já existe VPC e targets) ───────────────────
        alb = elbv2.ApplicationLoadBalancer(
            self, "ALB",
            vpc=vpc,
            internet_facing=True,
        )

        # ── 2. Trust store para mTLS (bundle de CAs em S3) ──────────────────
        ca_bucket = s3.Bucket.from_bucket_name(self, "CaBucket", "my-ca-bundles")
        trust_store = elbv2.TrustStore(
            self, "B2BTrustStore",
            bucket=ca_bucket,
            key="ca-bundle.pem",
        )

        # ── 3. Listener HTTPS com mTLS verify ────────────────────────────────
        listener = alb.add_listener(
            "HttpsListener",
            port=443,
            certificates=[certificate],
            mutual_authentication=elbv2.MutualAuthentication(
                mode=elbv2.MutualAuthenticationMode.VERIFY,
                trust_store=trust_store,
                ignore_client_certificate_expiry=False,
            ),
            default_action=elbv2.ListenerAction.fixed_response(
                status_code=403,
                content_type=elbv2.ContentType.TEXT_PLAIN,
                message_body="Forbidden",
            ),
        )

        # ── 4. Cognito User Pool ──────────────────────────────────────────────
        user_pool = cognito.UserPool(self, "UserPool",
            self_sign_up_enabled=True,
        )
        user_pool_client = user_pool.add_client("AlbClient",
            generate_secret=True,
            o_auth=cognito.OAuthSettings(
                flows=cognito.OAuthFlows(authorization_code_grant=True),
                scopes=[cognito.OAuthScope.OPENID, cognito.OAuthScope.EMAIL],
                callback_urls=[f"https://{alb.load_balancer_dns_name}/oauth2/idpresponse"],
            ),
        )
        user_pool_domain = user_pool.add_domain("Domain",
            cognito_domain=cognito.CognitoDomainOptions(domain_prefix="minha-app"),
        )

        # ── 5. Regra OIDC (prioridade 10): /dashboard/* → autentica com Cognito
        listener.add_action(
            "DashboardAuth",
            priority=10,
            conditions=[elbv2.ListenerCondition.path_patterns(["/dashboard/*"])],
            action=elbv2_actions.AuthenticateCognitoAction(
                user_pool=user_pool,
                user_pool_client=user_pool_client,
                user_pool_domain=user_pool_domain,
                session_timeout=Duration.hours(8),
                on_unauthenticated_request=elbv2.UnauthenticatedAction.AUTHENTICATE,
                next=elbv2.ListenerAction.forward([tg_web]),
            ),
        )

        # ── 6. Regra B2B (prioridade 20): /api/b2b/* → valida header mTLS e faz forward
        # O mTLS verify já acontece no handshake TLS — aqui apenas roteamos o tráfego
        # A regra pode ainda verificar o subject DN via http-header condition
        listener.add_action(
            "B2BRoute",
            priority=20,
            conditions=[elbv2.ListenerCondition.path_patterns(["/api/b2b/*"])],
            action=elbv2.ListenerAction.forward([tg_b2b]),
        )

        # ── 7. Default: tráfego restante → TG público (sem auth)
        # (já definido no default_action do listener; poderíamos trocar pelo tg_public)

        # ── 8. WAF Web ACL (REGIONAL) ─────────────────────────────────────────
        web_acl = wafv2.CfnWebACL(
            self, "WebACL",
            scope="REGIONAL",
            default_action=wafv2.CfnWebACL.DefaultActionProperty(allow={}),
            visibility_config=wafv2.CfnWebACL.VisibilityConfigProperty(
                cloud_watch_metrics_enabled=True,
                metric_name="WebACL",
                sampled_requests_enabled=True,
            ),
            rules=[
                # Regra 1: OWASP baseline (700 WCU)
                wafv2.CfnWebACL.RuleProperty(
                    name="AWSManagedCommon",
                    priority=0,
                    override_action=wafv2.CfnWebACL.OverrideActionProperty(none={}),
                    statement=wafv2.CfnWebACL.StatementProperty(
                        managed_rule_group_statement=wafv2.CfnWebACL.ManagedRuleGroupStatementProperty(
                            vendor_name="AWS",
                            name="AWSManagedRulesCommonRuleSet",
                        ),
                    ),
                    visibility_config=wafv2.CfnWebACL.VisibilityConfigProperty(
                        cloud_watch_metrics_enabled=True,
                        metric_name="AWSManagedCommon",
                        sampled_requests_enabled=True,
                    ),
                ),
                # Regra 2: SQL injection (200 WCU)
                wafv2.CfnWebACL.RuleProperty(
                    name="AWSManagedSQLi",
                    priority=1,
                    override_action=wafv2.CfnWebACL.OverrideActionProperty(none={}),
                    statement=wafv2.CfnWebACL.StatementProperty(
                        managed_rule_group_statement=wafv2.CfnWebACL.ManagedRuleGroupStatementProperty(
                            vendor_name="AWS",
                            name="AWSManagedRulesSQLiRuleSet",
                        ),
                    ),
                    visibility_config=wafv2.CfnWebACL.VisibilityConfigProperty(
                        cloud_watch_metrics_enabled=True,
                        metric_name="AWSManagedSQLi",
                        sampled_requests_enabled=True,
                    ),
                ),
                # Regra 3: Rate limiting custom (10 WCU)
                wafv2.CfnWebACL.RuleProperty(
                    name="RateLimit",
                    priority=2,
                    action=wafv2.CfnWebACL.RuleActionProperty(
                        block=wafv2.CfnWebACL.BlockActionProperty(
                            custom_response=wafv2.CfnWebACL.CustomResponseProperty(
                                response_code=429,
                            ),
                        ),
                    ),
                    statement=wafv2.CfnWebACL.StatementProperty(
                        rate_based_statement=wafv2.CfnWebACL.RateBasedStatementProperty(
                            limit=2000,           # requests per 5-minute window
                            aggregate_key_type="IP",
                        ),
                    ),
                    visibility_config=wafv2.CfnWebACL.VisibilityConfigProperty(
                        cloud_watch_metrics_enabled=True,
                        metric_name="RateLimit",
                        sampled_requests_enabled=True,
                    ),
                ),
            ],
        )

        # ── 9. Associar Web ACL ao ALB ─────────────────────────────────────────
        wafv2.CfnWebACLAssociation(
            self, "WebACLAssociation",
            resource_arn=alb.load_balancer_arn,
            web_acl_arn=web_acl.attr_arn,
        )
```

### CLI — comandos essenciais

**Criar trust store para mTLS:**
```bash
# Fazer upload do bundle de CAs para S3
aws s3 cp ca-bundle.pem s3://my-ca-bundles/ca-bundle.pem

# Criar trust store
aws elbv2 create-trust-store \
  --name b2b-trust-store \
  --ca-certificates-bundle-s3-bucket my-ca-bundles \
  --ca-certificates-bundle-s3-key ca-bundle.pem

# Saída: retorna TrustStoreArn
```

**Modificar listener existente para adicionar mTLS verify:**
```bash
aws elbv2 modify-listener \
  --listener-arn arn:aws:elasticloadbalancing:... \
  --mutual-authentication \
    Mode=verify,TrustStoreArn=arn:aws:elasticloadbalancing:...:truststore/b2b-trust-store/...,\
    IgnoreClientCertificateExpiry=false,AdvertiseTrustStoreCaNames=enabled
```

**Criar regra OIDC via CLI:**
```bash
# Arquivo actions.json
cat > actions.json << 'EOF'
[
  {
    "Type": "authenticate-cognito",
    "AuthenticateCognitoConfig": {
      "UserPoolArn": "arn:aws:cognito-idp:us-east-1:123456789:userpool/us-east-1_ABC",
      "UserPoolClientId": "abcdefg123456",
      "UserPoolDomain": "minha-app",
      "SessionCookieName": "AWSELBAuthSessionCookie-dashboard",
      "SessionTimeout": 28800,
      "Scope": "openid email",
      "OnUnauthenticatedRequest": "authenticate"
    },
    "Order": 1
  },
  {
    "Type": "forward",
    "TargetGroupArn": "arn:aws:elasticloadbalancing:...:targetgroup/web-tg/...",
    "Order": 2
  }
]
EOF

aws elbv2 create-rule \
  --listener-arn arn:aws:elasticloadbalancing:... \
  --priority 10 \
  --conditions '[{"Field":"path-pattern","PathPatternConfig":{"Values":["/dashboard/*"]}}]' \
  --actions file://actions.json
```

**Criar Web ACL WAF e associar ao ALB:**
```bash
# Criar Web ACL (REGIONAL)
aws wafv2 create-web-acl \
  --name "ecommerce-waf" \
  --scope REGIONAL \
  --region us-east-1 \
  --default-action Allow={} \
  --rules file://waf-rules.json \
  --visibility-config \
    SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=ecommerceWAF

# Associar ao ALB
aws wafv2 associate-web-acl \
  --web-acl-arn arn:aws:wafv2:us-east-1:123456789:regional/webacl/ecommerce-waf/... \
  --resource-arn arn:aws:elasticloadbalancing:us-east-1:123456789:loadbalancer/app/my-alb/...

# Verificar associação
aws wafv2 get-web-acl-for-resource \
  --resource-arn arn:aws:elasticloadbalancing:us-east-1:123456789:loadbalancer/app/my-alb/...
```

**Testar se um managed rule group está bloqueando tráfego legítimo (Count mode temporário):**
```bash
# Colocar regra em Count mode para testes
aws wafv2 update-web-acl \
  --name "ecommerce-waf" \
  --scope REGIONAL \
  --region us-east-1 \
  --id <web-acl-id> \
  --lock-token <lock-token> \
  --default-action Allow={} \
  --rules '[{
    "Name": "AWSManagedCommon",
    "Priority": 0,
    "OverrideAction": {"Count": {}},   ← Count ao invés de None
    "Statement": {"ManagedRuleGroupStatement": {"VendorName": "AWS", "Name": "AWSManagedRulesCommonRuleSet"}},
    "VisibilityConfig": {"SampledRequestsEnabled": true, "CloudWatchMetricsEnabled": true, "MetricName": "AWSManagedCommon"}
  }]' \
  --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=ecommerceWAF
```

---

## Armadilhas comuns

### Armadilha 1: OIDC só funciona em listeners HTTPS — e o ALB precisa de saída para o IdP

[FATO] A ação `authenticate-oidc` e `authenticate-cognito` são rejeitadas se configuradas em um
listener HTTP (porta 80). Isso é um erro de configuração frequente em ambientes de desenvolvimento
onde o ALB não tem certificado TLS. Além disso, para completar o fluxo OIDC, o próprio ALB precisa
fazer chamadas de saída para `TokenEndpoint` e `UserInfoEndpoint`. Se o ALB for interno (scheme
`internal`) ou se estiver em uma subnet privada sem NAT Gateway, essas chamadas falharão
silenciosamente e o usuário receberá HTTP 500. O diagnóstico é verificar os logs de acesso do ALB
— actions `authenticate-oidc` com resultado `AuthFailure` indicam problemas de conectividade do
ALB para o IdP.

### Armadilha 2: mTLS verify rejeita a conexão TLS — o backend nunca recebe o request

No modo mTLS Verify, a rejeição de um certificado de cliente inválido acontece no *handshake TLS*
— antes de qualquer header HTTP ser enviado. Isso significa que o backend não vê esses requests,
e não há como distinguir via listener rules entre "cliente sem certificado" e "cliente com
certificado inválido". O ALB simplesmente fecha a conexão TLS com um `handshake_failure`. Os
**connection logs** do ALB (não os access logs) registram essas falhas — habilite
`connection-log-enabled` no ALB para diagnosticá-las. No modo Passthrough, ao contrário, a
conexão TLS sempre é estabelecida e o backend recebe os headers com a cadeia de certificados para
validar por conta própria.

### Armadilha 3: WAF WCU budget e regras de override action vs rule action

Há uma distinção sutil entre `OverrideAction` e `Action` nas regras WAF que gera confusão. Regras
que referenciam **managed rule groups** ou **rule groups externos** usam `OverrideAction` (com
valores `None` = usa a ação definida dentro do grupo, ou `Count` = sobrescreve todas as ações do
grupo para Count). Regras **customizadas** (rate-based, byte-match, geo-match) usam `Action`
(com valores `Allow`, `Block`, `Count`, `CAPTCHA`, `Challenge`). Trocar os dois causa erros de
validação da API. Além disso, ao exceder o orçamento de 1.500 WCU por Web ACL, o deploy falha com
`WAFSubscriptionNotFoundException` ou erro de quota — sempre calcule o total de WCUs dos grupos
antes do deploy.

---

## Exercício de reflexão

Você está construindo uma plataforma SaaS multi-tenant. O ALB já tem OIDC com Cognito configurado
para autenticar usuários. Um parceiro enterprise solicita que sua aplicação suporte autenticação
machine-to-machine para integrações batch — ou seja, um sistema automatizado do parceiro precisa
chamar sua API `/api/partner/*` sem intervenção humana (sem browser, sem redirect para login).

O problema: OIDC Authorization Code Grant requer interação humana com o browser (redirecionamentos).
mTLS Verify poderia resolver o cenário B2B — mas o parceiro questiona se é possível misturar OIDC
e mTLS no mesmo ALB, e como garantir que clientes mTLS não precisem também passar pelo fluxo OIDC.

Descreva a arquitetura que você projetaria: como configurar o listener para que (a) requisições em
`/dashboard/*` passem por OIDC Cognito, (b) requisições em `/api/partner/*` exijam certificado
mTLS sem OIDC, e (c) o WAF proteja ambas as rotas. Considere a ordem de prioridade das regras,
a interação entre mTLS (que acontece no handshake TLS, antes das listener rules) e OIDC (que é
uma listener action), e como você validaria no backend que o certificado mTLS pertence ao parceiro
correto usando os headers `X-Amzn-Mtls-*`.

---

## Recursos para aprofundar

**1. Authenticate users using an Application Load Balancer**
URL: https://docs.aws.amazon.com/elasticloadbalancing/latest/application/listener-authenticate-users.html
O que você vai encontrar: fluxo de autenticação OIDC passo a passo (com diagrama), parâmetros
completos de `AuthenticateOidcActionConfig` e `AuthenticateCognitoActionConfig`, comportamento do
cookie de sessão, logout, e verificação de assinatura JWT dos headers `x-amzn-oidc-data`.
Por que é a fonte certa: documentação primária oficial da AWS com exemplos de CLI e JSON.

**2. Mutual authentication with TLS in Application Load Balancer**
URL: https://docs.aws.amazon.com/elasticloadbalancing/latest/application/mutual-authentication.html
O que você vai encontrar: comparação entre modos passthrough e verify, requisitos do bundle PEM,
tabela completa de headers `X-Amzn-Mtls-*`, Advertise CA Subject Names, e referência para
connection logs.
Por que é a fonte certa: documentação primária com exemplos de headers reais e especificações
exatas de formatos aceitos.

**3. Configuring mutual TLS on an Application Load Balancer**
URL: https://docs.aws.amazon.com/elasticloadbalancing/latest/application/configuring-mtls-with-elb.html
O que você vai encontrar: passo a passo de configuração do trust store, upload do bundle para S3,
associação ao listener via console e CLI.
Por que é a fonte certa: guia procedimental complementar ao conceitual do link anterior.

**4. Associating or disassociating a web ACL with an AWS resource**
URL: https://docs.aws.amazon.com/waf/latest/developerguide/web-acl-associating-aws-resource.html
O que você vai encontrar: procedimento de associação de Web ACL a ALB, restrição de uma ACL por
recurso, requisito de escopo REGIONAL, e comportamento quando a ACL é dissociada.
Por que é a fonte certa: documentação primária da funcionalidade de associação WAF + ALB.

**5. AWS Managed Rules for AWS WAF**
URL: https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups.html
O que você vai encontrar: catálogo completo de managed rule groups com WCU de cada um, descrição
das ameaças cobertas, e changelog de atualizações (importante para entender quando regras novas
podem causar falsos positivos em produção).
Por que é a fonte certa: referência essencial para planejar o orçamento de WCUs e escolher os
grupos certos para o perfil de ameaças da sua aplicação.

---

