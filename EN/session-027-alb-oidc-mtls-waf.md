# Session 027 — ALB: native OIDC, mTLS and WAF integration

**Estimated duration:** 60 minutes
**Prerequisites:** session-026-alb-listener-rules-weighted

---

## Objective

By the end, you will be able to configure native OIDC authentication on the ALB (delegating auth to Cognito or any OIDC IdP without backend code), enable mTLS with a CA trust store, and associate a WAF Web ACL to an ALB to filter requests before they reach the application.

---

## Context

[FACT] The ALB supports three security mechanisms at the load balancer level — each acts on a different layer of the threat model and complements the others. Native OIDC delegates to the ALB the role of authenticating human users via OAuth2/OIDC flow, eliminating the need to implement this flow in each microservice. mTLS goes beyond conventional TLS and requires that the *client* present an X.509 certificate signed by a trusted CA — useful for machine-to-machine authentication at service boundaries. WAF (Web Application Firewall) acts even before the request reaches the listener, filtering malicious traffic based on L7 rules (SQL injection, XSS, rate limiting, geo-blocking).

[CONSENSUS] The combination of the three mechanisms follows the defense in depth principle: WAF blocks known threats at the perimeter, mTLS ensures only authorized clients establish TLS connections, and OIDC ensures only authenticated users reach the backend. Each layer can fail or be bypassed in different ways — having them together significantly reduces the attack surface.

---

## Core concepts

### 1. Native OIDC on ALB — how the authentication flow works

[FACT] The ALB supports two types of authentication actions in HTTPS listener rules: `authenticate-oidc` (for any IdP that implements OIDC) and `authenticate-cognito` (shortcut for Cognito User Pools integration). Both follow the OAuth 2.0 Authorization Code Grant flow. Only HTTPS listeners support these actions — on HTTP listeners the action is rejected.

The authentication flow has 11 steps documented by AWS:

```
Client                      ALB                         IdP (OIDC)          Backend
  |                           |                              |                   |
  |--- HTTPS request -------->|                              |                   |
  |                           |-- checks AWSELB cookie ----->|                   |
  |                           |   (doesn't exist 1st time)  |                   |
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
  |--- original request ------>|                             |                   |
  |                           |-- forward + X-AMZN-OIDC-* headers ------------->|
  |<-- response -------------------------------------------------- response -----|
```

[FACT] The session cookie `AWSELB` is sent to the client after successful authentication. Subsequent requests present this cookie and the ALB validates locally (without roundtrip to IdP) — jumping directly to step 9. The cookie always carries the `Secure` attribute (requires HTTPS). For CORS requests, it also includes `SameSite=None`.

[FACT] If total user claims + access token exceed 11 KB, the ALB returns HTTP 500 and increments the `ELBAuthUserClaimsSizeExceeded` metric. Cookies larger than 4 KB are fragmented into multiple cookies (the ALB supports up to 4 shards, totaling up to 16 KB).

**Headers sent to backend:**

| Header | Content |
|--------|---------|
| `x-amzn-oidc-accesstoken` | Access token in plain text |
| `x-amzn-oidc-identity` | `sub` claim from user info endpoint (plain text) |
| `x-amzn-oidc-data` | User claims in JWT format (ES256, base64url encoded) |

[FACT] The `sub` field is the canonical way to identify a user — `sub` is stable and unique per IdP, while fields like `email` can change. The JWT in `x-amzn-oidc-data` is signed by the ALB with ES256 (ECDSA P-256 + SHA256). The public key for verification is available at: `https://public-keys.auth.elb.<region>.amazonaws.com/<key-id>` (the `kid` is in the JWT header).

**`OnUnauthenticatedRequest` behavior:**

```
authenticate  → redirects to IdP (default; good for SPAs and traditional web apps)
allow         → passes request without claims (good for apps with public + private content)
deny          → returns HTTP 401 (good for APIs that fetch in background — avoids redirect loop)
```

**Session timeout:** default 7 days, configurable from 1 second to 7 days. Independent of cookie TTL (always 7 days in `Max-Age` attribute) — the real timeout is encrypted within the cookie value.

**Client login timeout:** fixed at 15 minutes. The user must complete login at the IdP within this window; otherwise, receives HTTP 401 and needs to reload the page.

### 2. OIDC configuration — required and optional parameters

[FACT] For `authenticate-oidc`, the following fields are required in the action:

```
Issuer              — IdP issuer URL (e.g.: https://accounts.google.com)
AuthorizationEndpoint — Authorization URL (e.g.: https://accounts.google.com/o/oauth2/v2/auth)
TokenEndpoint       — Token endpoint URL (e.g.: https://oauth2.googleapis.com/token)
UserInfoEndpoint    — User info URL (e.g.: https://openidconnect.googleapis.com/v1/userinfo)
ClientId            — OAuth 2.0 client ID (registered at the IdP)
ClientSecret        — OAuth 2.0 client secret
```

[FACT] DNS of IdP endpoints must be publicly resolvable (even if resolving to private IPs). The ALB communicates with `TokenEndpoint` and `UserInfoEndpoint` — if the ALB is internal or uses `dualstack-without-public-ipv4`, a NAT Gateway is needed for the ALB to reach these endpoints. The ALB only supports IPv4 in this communication.

[FACT] The callback URL that must be allowed at the IdP follows the pattern: `https://<ALB-DNS>/oauth2/idpresponse` or `https://<CNAME>/oauth2/idpresponse`.

**Relevant optional parameters:**

```
SessionCookieName         — cookie name (default: AWSELBAuthSessionCookie)
                            use distinct names for multiple rules with auth on the same listener
SessionTimeout            — session time in seconds (default: 604800 = 7 days)
Scope                     — OAuth scopes requested (default: "openid"; add "email" for email claims)
AuthenticationRequestExtraParams — extra parameters for the IdP (e.g.: {"prompt": "login"})
```

**For Cognito (`authenticate-cognito`), the required fields are:**
```
UserPoolArn        — User Pool ARN
UserPoolClientId   — App client ID (must have client secret + code grant enabled)
UserPoolDomain     — User Pool domain (e.g.: my-app.auth.us-east-1.amazoncognito.com)
```

### 3. mTLS on ALB — two modes and trust store

[FACT] Mutual TLS (mTLS) inverts conventional TLS authentication: in addition to the server presenting its certificate to the client, the *client* must present an X.509v3 certificate signed by a CA that the ALB recognizes as trusted. This is especially relevant in B2B scenarios, machine-to-machine, and zero-trust between services.

The ALB offers two mTLS modes:

```
┌─────────────────────────────────────────────────────────────────────┐
│                     mTLS PASSTHROUGH                                 │
│                                                                      │
│  Client → [cert chain] → ALB → [X-Amzn-Mtls-Clientcert header] → Backend │
│                                                                      │
│  The ALB does NOT verify the client certificate.                     │
│  Forwards the complete chain (URL-encoded PEM) in the header.        │
│  The backend is responsible for validating and authorizing.          │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                      mTLS VERIFY                                     │
│                                                                      │
│  Client → [cert] → ALB → validates against trust store → forward    │
│                           (rejects if cert invalid/not trusted)      │
│                                                                      │
│  The ALB verifies the client's X.509 chain against the trust store.  │
│  Sends cert metadata to backend via X-Amzn-Mtls-* headers.          │
└─────────────────────────────────────────────────────────────────────┘
```

[FACT] For Verify mode, a **trust store** resource must be created — a collection of root and intermediate CA certificates. The CA bundle is stored in S3 in PEM format and associated with the trust store. To replace the bundle, use the `ModifyTrustStore` API — it's not possible to add individual certificates, only replace the entire bundle.

**Headers sent to backend in Verify mode:**

```
X-Amzn-Mtls-Clientcert-Serial-Number  → hexadecimal serial number of leaf cert
X-Amzn-Mtls-Clientcert-Issuer         → issuer DN (RFC 2253)
X-Amzn-Mtls-Clientcert-Subject        → subject DN (RFC 2253)
X-Amzn-Mtls-Clientcert-Validity       → NotBefore=...;NotAfter=... (ISO 8601)
X-Amzn-Mtls-Clientcert-Leaf           → leaf cert in URL-encoded PEM
```

**Header in Passthrough mode:**
```
X-Amzn-Mtls-Clientcert  → complete chain in URL-encoded PEM (leaf → root)
```

[FACT] Session resumption (TLS session tickets) is not supported in mTLS passthrough or verify modes. Each new TLS connection performs the full handshake.

### 4. AWS WAF — architecture and ALB integration

[FACT] AWS WAF v2 operates as an L7 "inspector" proxy that evaluates the HTTP/HTTPS request *before* it reaches the ALB target. The configuration unit is the **Web ACL** (Access Control List), which contains rules ordered by priority. WAF evaluates rules in ascending priority order and applies the action of the first rule that matches.

```
Internet
    │
    ▼
┌─────────────┐
│  AWS WAF    │  ← Web ACL (REGIONAL scope)
│  Web ACL    │     Rule 1 (prio 0): AWSManagedRulesCommonRuleSet
│             │     Rule 2 (prio 1): AWSManagedRulesSQLiRuleSet
│             │     Rule 3 (prio 2): custom rate-limit rule
│             │     Default action: Allow
└──────┬──────┘
       │  (only what wasn't blocked passes through)
       ▼
┌─────────────┐
│     ALB     │  ← listener rules, OIDC, mTLS
└──────┬──────┘
       │
       ▼
   Targets (ECS, EC2, Lambda...)
```

[FACT] To use WAF with ALB, the Web ACL must be created with scope `REGIONAL` (not `CLOUDFRONT`). Each AWS resource can only be associated with **one** Web ACL at a time. WAF and ALB must be in the same region.

**Possible actions in each WAF rule:**

```
Allow   → passes the request; terminates evaluation (no other rule is executed)
Block   → returns HTTP 403 (default) or custom response; terminates evaluation
Count   → increments counter; does NOT terminate evaluation (continues to next rules)
CAPTCHA → presents CAPTCHA challenge to user before proceeding
Challenge → verifies if it's a real browser (silent JavaScript challenge)
```

[FACT] `Count` is especially useful for testing new rules in production without blocking traffic — you monitor the `CountedRequests` metrics in CloudWatch before switching to `Block`.

**Relevant AWS Managed Rule Groups for ALB:**

| Group | What it covers |
|-------|----------------|
| `AWSManagedRulesCommonRuleSet` | OWASP Top 10 baseline (XSS, LFI, RFI, SSRF) |
| `AWSManagedRulesSQLiRuleSet` | SQL injection in query strings, body, headers |
| `AWSManagedRulesKnownBadInputsRuleSet` | Malformed inputs, Log4JRCE, Spring4Shell |
| `AWSManagedRulesAmazonIpReputationList` | IPs with bad reputation (bots, scrapers, TOR) |
| `AWSManagedRulesAnonymousIpList` | Anonymous proxies, VPNs, TOR exit nodes |
| `AWSManagedRulesBotControlRuleSet` | Bot detection (pay-per-use; charges extra WCU) |

[FACT] Each Web ACL has a quota of **WCU (WAF Capacity Units)**. The default limit is 1,500 WCU per Web ACL. `AWSManagedRulesCommonRuleSet` consumes 700 WCU, `AWSManagedRulesSQLiRuleSet` consumes 200 WCU. When combining multiple groups, WCU budget planning is necessary.

[FACT] The WAF → ALB association uses the ALB ARN. Once associated, WAF inspects all requests that arrive at the ALB — it's not possible to select specific listeners to inspect.

---

## Practical example

**Scenario:** E-commerce API with three security requirements:
1. Web users authenticate via Cognito (OIDC) before accessing `/dashboard/*`
2. B2B integration (partners) authenticates via X.509 certificate (mTLS verify) at `/api/b2b/*`
3. WAF protects the entire ALB against OWASP Top 10 and SQL injection

### CLI — essential commands

**Create trust store for mTLS:**
```bash
# Upload CA bundle to S3
aws s3 cp ca-bundle.pem s3://my-ca-bundles/ca-bundle.pem

# Create trust store
aws elbv2 create-trust-store \
  --name b2b-trust-store \
  --ca-certificates-bundle-s3-bucket my-ca-bundles \
  --ca-certificates-bundle-s3-key ca-bundle.pem

# Output: returns TrustStoreArn
```

**Modify existing listener to add mTLS verify:**
```bash
aws elbv2 modify-listener \
  --listener-arn arn:aws:elasticloadbalancing:... \
  --mutual-authentication \
    Mode=verify,TrustStoreArn=arn:aws:elasticloadbalancing:...:truststore/b2b-trust-store/...,\
    IgnoreClientCertificateExpiry=false,AdvertiseTrustStoreCaNames=enabled
```

**Create OIDC rule via CLI:**
```bash
# actions.json file
cat > actions.json << 'EOF'
[
  {
    "Type": "authenticate-cognito",
    "AuthenticateCognitoConfig": {
      "UserPoolArn": "arn:aws:cognito-idp:us-east-1:123456789:userpool/us-east-1_ABC",
      "UserPoolClientId": "abcdefg123456",
      "UserPoolDomain": "my-app",
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

**Create WAF Web ACL and associate with ALB:**
```bash
# Create Web ACL (REGIONAL)
aws wafv2 create-web-acl \
  --name "ecommerce-waf" \
  --scope REGIONAL \
  --region us-east-1 \
  --default-action Allow={} \
  --rules file://waf-rules.json \
  --visibility-config \
    SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=ecommerceWAF

# Associate with ALB
aws wafv2 associate-web-acl \
  --web-acl-arn arn:aws:wafv2:us-east-1:123456789:regional/webacl/ecommerce-waf/... \
  --resource-arn arn:aws:elasticloadbalancing:us-east-1:123456789:loadbalancer/app/my-alb/...

# Verify association
aws wafv2 get-web-acl-for-resource \
  --resource-arn arn:aws:elasticloadbalancing:us-east-1:123456789:loadbalancer/app/my-alb/...
```

---

## Common pitfalls

### Pitfall 1: OIDC only works on HTTPS listeners — and the ALB needs outbound to the IdP

[FACT] The `authenticate-oidc` and `authenticate-cognito` actions are rejected if configured on an HTTP listener (port 80). This is a frequent configuration error in development environments where the ALB doesn't have a TLS certificate. Additionally, to complete the OIDC flow, the ALB itself needs to make outbound calls to `TokenEndpoint` and `UserInfoEndpoint`. If the ALB is internal (scheme `internal`) or is in a private subnet without NAT Gateway, these calls will fail silently and the user will receive HTTP 500. Diagnosis is checking the ALB access logs — `authenticate-oidc` actions with result `AuthFailure` indicate ALB-to-IdP connectivity problems.

### Pitfall 2: mTLS verify rejects the TLS connection — the backend never receives the request

In mTLS Verify mode, rejection of an invalid client certificate happens at the *TLS handshake* — before any HTTP header is sent. This means the backend doesn't see these requests, and there's no way to distinguish via listener rules between "client without certificate" and "client with invalid certificate". The ALB simply closes the TLS connection with a `handshake_failure`. The ALB's **connection logs** (not access logs) record these failures — enable `connection-log-enabled` on the ALB to diagnose them. In Passthrough mode, conversely, the TLS connection is always established and the backend receives the headers with the certificate chain to validate on its own.

### Pitfall 3: WAF WCU budget and override action vs rule action rules

There's a subtle distinction between `OverrideAction` and `Action` in WAF rules that generates confusion. Rules that reference **managed rule groups** or **external rule groups** use `OverrideAction` (with values `None` = uses the action defined within the group, or `Count` = overrides all group actions to Count). **Custom** rules (rate-based, byte-match, geo-match) use `Action` (with values `Allow`, `Block`, `Count`, `CAPTCHA`, `Challenge`). Swapping the two causes API validation errors. Additionally, exceeding the 1,500 WCU budget per Web ACL causes deploy failure with `WAFSubscriptionNotFoundException` or quota error — always calculate the total WCUs of the groups before deploying.

---

## Reflection exercise

You are building a multi-tenant SaaS platform. The ALB already has OIDC with Cognito configured to authenticate users. An enterprise partner requests that your application support machine-to-machine authentication for batch integrations — meaning an automated system from the partner needs to call your API `/api/partner/*` without human intervention (no browser, no redirect to login).

The problem: OIDC Authorization Code Grant requires human interaction with the browser (redirects). mTLS Verify could solve the B2B scenario — but the partner questions whether it's possible to mix OIDC and mTLS on the same ALB, and how to ensure mTLS clients don't also need to go through the OIDC flow.

Describe the architecture you would design: how to configure the listener so that (a) requests to `/dashboard/*` go through OIDC Cognito, (b) requests to `/api/partner/*` require mTLS certificate without OIDC, and (c) WAF protects both routes. Consider the rule priority order, the interaction between mTLS (which happens at the TLS handshake, before listener rules) and OIDC (which is a listener action), and how you would validate in the backend that the mTLS certificate belongs to the correct partner using the `X-Amzn-Mtls-*` headers.

---

## Resources for further study

**1. Authenticate users using an Application Load Balancer**
URL: https://docs.aws.amazon.com/elasticloadbalancing/latest/application/listener-authenticate-users.html
What you'll find: OIDC authentication flow step by step (with diagram), complete parameters of `AuthenticateOidcActionConfig` and `AuthenticateCognitoActionConfig`, session cookie behavior, logout, and JWT signature verification of `x-amzn-oidc-data` headers.
Why it's the right source: official primary AWS documentation with CLI and JSON examples.

**2. Mutual authentication with TLS in Application Load Balancer**
URL: https://docs.aws.amazon.com/elasticloadbalancing/latest/application/mutual-authentication.html
What you'll find: comparison between passthrough and verify modes, PEM bundle requirements, complete table of `X-Amzn-Mtls-*` headers, Advertise CA Subject Names, and reference for connection logs.
Why it's the right source: primary documentation with real header examples and exact specifications of accepted formats.

**3. Configuring mutual TLS on an Application Load Balancer**
URL: https://docs.aws.amazon.com/elasticloadbalancing/latest/application/configuring-mtls-with-elb.html
What you'll find: step-by-step trust store configuration, bundle upload to S3, listener association via console and CLI.
Why it's the right source: procedural guide complementary to the conceptual link above.

**4. Associating or disassociating a web ACL with an AWS resource**
URL: https://docs.aws.amazon.com/waf/latest/developerguide/web-acl-associating-aws-resource.html
What you'll find: Web ACL to ALB association procedure, one ACL per resource restriction, REGIONAL scope requirement, and behavior when ACL is disassociated.
Why it's the right source: primary documentation of WAF + ALB association functionality.

**5. AWS Managed Rules for AWS WAF**
URL: https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups.html
What you'll find: complete catalog of managed rule groups with WCU for each, description of covered threats, and update changelog (important for understanding when new rules can cause false positives in production).
Why it's the right source: essential reference for planning WCU budget and choosing the right groups for your application's threat profile.

---
