# Sessão session-025 — Lambda@Edge vs CloudFront Functions: casos de uso e limites

**Duração estimada:** 60 minutos
**Pré-requisitos:** session-024-lambda-observabilidade-xray-insights

---

## Objetivo

Ao final, você conseguirá escolher entre Lambda@Edge e CloudFront Functions para um requisito dado (baseado em latência de execução, acesso ao body, número de regiões, custo e tempo de deploy), implementar um header injection simples com CloudFront Functions, e entender os limites de timeout e memória de cada opção.

---

## Contexto

[FATO] CloudFront é a CDN (Content Delivery Network) da AWS. Ela distribui conteúdo a partir de mais de 600 pontos de presença (PoPs) ao redor do mundo, chamados **edge locations**. Quando uma requisição chega ao CloudFront, ela é processada no edge location mais próximo do usuário — não na região AWS da origem. Isso significa que qualquer lógica executada no edge tem latência radicalmente menor do que uma chamada de volta à origem.

[FATO] Dois serviços permitem executar código no edge do CloudFront: **CloudFront Functions** (lançado em 2021) e **Lambda@Edge** (lançado em 2017). Eles diferem fundamentalmente em capacidade, latência, custo e casos de uso. Não são concorrentes diretos — cada um resolve uma classe diferente de problemas.

[CONSENSO] A regra geral adotada pela maioria das arquiteturas de produção: se a lógica é simples (manipulação de headers, URL rewrites, normalização de cache keys) e roda em cada request, use CloudFront Functions. Se a lógica é complexa (acesso ao body, chamadas a serviços externos, autenticação JWT completa, decisões baseadas em banco de dados), use Lambda@Edge. O custo de CloudFront Functions é aproximadamente 1/6 do Lambda@Edge por requisição.

---

## Conceitos principais

### 1. Os quatro pontos de interceptação do CloudFront

[FATO] O CloudFront intercepta o fluxo HTTP em quatro pontos distintos. Cada ponto tem características diferentes e suporta diferentes tipos de edge function:

```
Usuário
   │
   ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                         CLOUDFRONT EDGE LOCATION                            │
│                                                                              │
│  ① viewer-request                              ② viewer-response            │
│  Antes do cache check                          Antes de retornar ao usuário  │
│  (CloudFront Functions ✓)                      (CloudFront Functions ✓)      │
│  (Lambda@Edge ✓)                               (Lambda@Edge ✓)              │
│         │                                              ▲                    │
│         ▼                                              │                    │
│   ┌──────────────┐                           ┌─────────────────┐           │
│   │  Cache hit?  │──── HIT ─────────────────►│ Serve do cache  │           │
│   └──────┬───────┘                           └─────────────────┘           │
│          │ MISS                                                              │
│          ▼                                                                   │
│  ③ origin-request            ④ origin-response                             │
│  Antes de ir à origem        Após receber da origem (e cachear)              │
│  (Lambda@Edge ✓ apenas)      (Lambda@Edge ✓ apenas)                         │
│         │                                    ▲                              │
└─────────┼────────────────────────────────────┼──────────────────────────────┘
          │                                    │
          ▼                                    │
       ORIGEM (S3, ALB, EC2, API Gateway...)───┘
```

[FATO] **CloudFront Functions** só pode ser associado a `viewer-request` e `viewer-response`. **Lambda@Edge** pode ser associado aos quatro eventos. Essa distinção é fundamental: apenas Lambda@Edge pode interceptar e modificar requests/responses para a origem.

---

### 2. CloudFront Functions — sub-millisegundo, escala massiva, limitações estritas

[FATO] CloudFront Functions executa código JavaScript (ES2015+) em um runtime **próprio** — não é Node.js. É uma engine JavaScript leve, sem módulos npm, sem acesso à rede, sem timers assíncronos, sem `require()`. O código deve ser **síncrono** e terminar em menos de 1ms de compute time.

#### Limites e capacidades

[FATO] Limites documentados:

```
┌──────────────────────────────┬──────────────────────────────────────────┐
│ Limite                       │ Valor                                    │
├──────────────────────────────┼──────────────────────────────────────────┤
│ Tempo máximo de execução     │ ~1ms (compute utilization 0-100)        │
│ Memória                      │ 2 MB                                     │
│ Tamanho máximo do código     │ 10 KB                                    │
│ Runtime                      │ JavaScript (ES2015+) — NÃO é Node.js    │
│ Acesso ao body da requisição │ NÃO                                      │
│ Chamadas de rede             │ NÃO                                      │
│ Variáveis de ambiente        │ NÃO (use Key Value Store)                │
│ Acesso a sistema de arquivos │ NÃO                                      │
│ Eventos suportados           │ viewer-request, viewer-response          │
│ Escala                       │ Milhões de req/s instantaneamente        │
│ Deploy                       │ CloudFront nativo (não via Lambda)       │
│ Teste integrado              │ Console CloudFront                       │
└──────────────────────────────┴──────────────────────────────────────────┘
```

#### Estrutura do evento (CloudFront Functions)

[FATO] O objeto de evento tem estrutura diferente do Lambda@Edge:

```javascript
// evento recebido pela função (viewer-request)
{
  "version": "1.0",
  "context": {
    "distributionDomainName": "d111111abcdef8.cloudfront.net",
    "distributionId": "EDFDVBD6EXAMPLE",
    "eventType": "viewer-request",
    "requestId": "4TyzHTaYWb1GX1qTfsHhEqV6HUDd_BzoBZnwfnvQc_1oF26ClkoUSEQ=="
  },
  "viewer": {
    "ip": "1.2.3.4"
  },
  "request": {
    "method": "GET",
    "uri": "/index.html",
    "querystring": {
      "cat": { "value": "meow" }
    },
    "headers": {
      "host": { "value": "www.example.com" },
      "accept-language": { "value": "pt-BR,pt;q=0.9" }
    },
    "cookies": {
      "session": { "value": "abc123" }
    }
  }
}
```

[FATO] A função deve retornar o objeto `request` (para viewer-request) ou `response` (para viewer-response). Se retornar um objeto `response` em viewer-request, a requisição é **interrompida** — nunca chega ao cache ou à origem.

#### Exemplos de CloudFront Functions

```javascript
// 1. Injetar Security Headers (viewer-response)
function handler(event) {
    var response = event.response;
    var headers = response.headers;

    headers['strict-transport-security'] = { value: 'max-age=31536000; includeSubdomains; preload' };
    headers['x-content-type-options'] = { value: 'nosniff' };
    headers['x-frame-options'] = { value: 'DENY' };
    headers['x-xss-protection'] = { value: '1; mode=block' };
    headers['referrer-policy'] = { value: 'strict-origin-when-cross-origin' };
    headers['permissions-policy'] = { value: 'camera=(), microphone=(), geolocation=()' };

    return response;
}
```

```javascript
// 2. Normalizar cache key — remover query strings irrelevantes (viewer-request)
// Sem isso, "?utm_source=email" e "?utm_source=social" geram cache misses separados
function handler(event) {
    var request = event.request;
    var qs = request.querystring;

    // Remove parâmetros de tracking — não afetam o conteúdo
    var tracking = ['utm_source', 'utm_medium', 'utm_campaign', 'utm_content', 'fbclid', 'gclid'];
    tracking.forEach(function(param) { delete qs[param]; });

    return request;
}
```

```javascript
// 3. URL rewrite — SPA com todas as rotas servindo index.html (viewer-request)
function handler(event) {
    var request = event.request;
    var uri = request.uri;

    // Se não tem extensão de arquivo, serve index.html
    if (!uri.includes('.')) {
        request.uri = '/index.html';
    }
    return request;
}
```

```javascript
// 4. Redirect HTTP → HTTPS e www → apex (viewer-request)
function handler(event) {
    var request = event.request;
    var headers = request.headers;
    var host = headers.host ? headers.host.value : '';

    // Redireciona www para apex
    if (host.startsWith('www.')) {
        return {
            statusCode: 301,
            statusDescription: 'Moved Permanently',
            headers: {
                'location': { value: 'https://' + host.slice(4) + request.uri }
            }
        };
    }

    return request;
}
```

```javascript
// 5. A/B testing via cookie (viewer-request)
// Atribui usuários novos a uma variante e redireciona para o path correto
function handler(event) {
    var request = event.request;
    var cookies = request.cookies;

    // Usuário já tem variante atribuída
    if (cookies['ab-variant']) {
        var variant = cookies['ab-variant'].value;
        request.uri = '/' + variant + request.uri;
        return request;
    }

    // Atribui nova variante (50/50)
    var variant = Math.random() < 0.5 ? 'a' : 'b';
    request.uri = '/' + variant + request.uri;

    // Retorna response para definir o cookie — próximo request não será atribuído
    return {
        statusCode: 302,
        statusDescription: 'Found',
        headers: {
            'location': { value: request.uri },
            'set-cookie': { value: 'ab-variant=' + variant + '; Path=/; Max-Age=2592000' }
        }
    };
}
```

---

### 3. Lambda@Edge — poder completo, complexidade operacional maior

[FATO] Lambda@Edge são funções Lambda comuns, com as seguintes restrições específicas para execução no edge:

```
┌─────────────────────────────────┬──────────────────────┬──────────────────────┐
│ Limite                          │ Viewer events        │ Origin events         │
├─────────────────────────────────┼──────────────────────┼──────────────────────┤
│ Timeout máximo                  │ 5 segundos           │ 30 segundos           │
├─────────────────────────────────┼──────────────────────┼──────────────────────┤
│ Memória máxima                  │ 128 MB               │ 128 MB – 10.240 MB    │
├─────────────────────────────────┼──────────────────────┼──────────────────────┤
│ Tamanho do package (comprimido) │ 1 MB                 │ 50 MB                 │
├─────────────────────────────────┼──────────────────────┼──────────────────────┤
│ Acesso ao body                  │ Sim (40 KB)          │ Sim (1 MB)            │
├─────────────────────────────────┼──────────────────────┼──────────────────────┤
│ Chamadas de rede                │ Sim                  │ Sim                   │
├─────────────────────────────────┼──────────────────────┼──────────────────────┤
│ Variáveis de ambiente           │ NÃO                  │ NÃO                   │
├─────────────────────────────────┼──────────────────────┼──────────────────────┤
│ Layers Lambda                   │ NÃO                  │ NÃO                   │
├─────────────────────────────────┼──────────────────────┼──────────────────────┤
│ Runtime                         │ Node.js, Python      │ Node.js, Python       │
├─────────────────────────────────┼──────────────────────┼──────────────────────┤
│ Deploy                          │ us-east-1 apenas     │ us-east-1 apenas      │
└─────────────────────────────────┴──────────────────────┴──────────────────────┘
```

[FATO] Lambda@Edge **deve ser deployada na região us-east-1**. A replicação para todos os edge locations do mundo é feita automaticamente pelo CloudFront ao associar a função a uma distribuição. Você gerencia apenas a função em us-east-1.

[FATO] Lambda@Edge **não suporta variáveis de ambiente**. Configurações que normalmente iriam em environment variables devem ser hardcoded, buscar de SSM Parameter Store/Secrets Manager em runtime (com cache no init phase), ou injetadas via CloudFormation ao fazer o deploy.

[FATO] A **execution role** da função Lambda@Edge deve ter como trusted principal tanto `lambda.amazonaws.com` quanto `edgelambda.amazonaws.com`:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Service": ["lambda.amazonaws.com", "edgelambda.amazonaws.com"]
    },
    "Action": "sts:AssumeRole"
  }]
}
```

#### Estrutura do evento (Lambda@Edge)

[FATO] O evento tem uma estrutura mais verbosa — os dados estão aninhados em `Records[0].cf`:

```python
# viewer-request event (Python)
{
  "Records": [{
    "cf": {
      "config": {
        "distributionDomainName": "d111111abcdef8.cloudfront.net",
        "distributionId": "EDFDVBD6EXAMPLE",
        "eventType": "viewer-request",
        "requestId": "4TyzHTa..."
      },
      "request": {
        "clientIp": "203.0.113.178",
        "method": "GET",
        "uri": "/picture.jpg",
        "querystring": "size=large&color=red",
        "headers": {
          "host": [{ "key": "Host", "value": "d111111abcdef8.cloudfront.net" }],
          "accept-language": [{ "key": "Accept-Language", "value": "en-US,en;q=0.5" }]
        },
        "body": {  # apenas se include_body=True e método POST/PUT
          "inputTruncated": False,
          "action": "read-only",
          "encoding": "base64",
          "data": "aGVsbG8="
        }
      }
    }
  }]
}
```

[FATO] A função Lambda@Edge deve retornar um objeto com a mesma estrutura do `request` ou `response` (sem o wrapper `Records`). Para interromper a requisição, retorne um objeto `response`:

```python
# Exemplo: autenticação JWT em viewer-request (Lambda@Edge)
import json
import base64
import hmac
import hashlib

# Cache da chave pública (inicializado no init phase, reusado em warm starts)
_JWT_SECRET = None

def get_secret():
    global _JWT_SECRET
    if _JWT_SECRET is None:
        import boto3
        # NOTA: Lambda@Edge em us-east-1, SSM deve estar em us-east-1 também
        ssm = boto3.client('ssm', region_name='us-east-1')
        _JWT_SECRET = ssm.get_parameter(
            Name='/app/jwt-secret', WithDecryption=True
        )['Parameter']['Value']
    return _JWT_SECRET

def handler(event, context):
    request = event['Records'][0]['cf']['request']
    headers = request['headers']

    # Extrai token do header Authorization
    auth_header = headers.get('authorization', [{}])[0].get('value', '')
    if not auth_header.startswith('Bearer '):
        return unauthorized_response()

    token = auth_header[7:]

    try:
        payload = verify_jwt(token, get_secret())
        # Injeta user_id como header para a origem
        request['headers']['x-user-id'] = [{
            'key': 'X-User-Id',
            'value': payload['sub']
        }]
        return request
    except Exception:
        return unauthorized_response()

def verify_jwt(token, secret):
    parts = token.split('.')
    if len(parts) != 3:
        raise ValueError("Invalid JWT")
    header_payload = f"{parts[0]}.{parts[1]}"
    signature = base64.urlsafe_b64decode(parts[2] + '==')
    expected = hmac.new(secret.encode(), header_payload.encode(), hashlib.sha256).digest()
    if not hmac.compare_digest(signature, expected):
        raise ValueError("Invalid signature")
    payload = json.loads(base64.urlsafe_b64decode(parts[1] + '=='))
    return payload

def unauthorized_response():
    return {
        'status': '401',
        'statusDescription': 'Unauthorized',
        'headers': {
            'www-authenticate': [{ 'key': 'WWW-Authenticate', 'value': 'Bearer' }],
            'content-type': [{ 'key': 'Content-Type', 'value': 'application/json' }]
        },
        'body': json.dumps({ 'error': 'Unauthorized' })
    }
```

---

### 4. Tabela de decisão — CloudFront Functions vs Lambda@Edge

[FATO] A documentação oficial da AWS resume a decisão em:

```
USE CloudFront Functions quando:             USE Lambda@Edge quando:
──────────────────────────────────────────   ──────────────────────────────────────────
✓ URL normalization e rewrites               ✓ Autenticação/autorização complexa
✓ Manipulação de headers (add/remove/mod)    ✓ Lógica dependente do body da requisição
✓ Normalizar cache keys (remover params)     ✓ Chamadas a AWS (DynamoDB, SSM, etc.)
✓ Cookie manipulation simples                ✓ Resposta diferente por geolocalização
✓ A/B redirect por cookie                   ✓ Processamento de imagens
✓ Redirecionar HTTP → HTTPS                  ✓ Geração de respostas dinâmicas
✓ Injetar security headers                  ✓ Reescrita de URL baseada em dados externos
✓ Custo mínimo (milhões de req/s)           ✓ Interceptar origin-request/response
✓ Deploy instantâneo                        ✓ Modificar request antes de ir à S3/ALB
```

**Custo comparativo (aproximado):**

```
CloudFront Functions:
  $0.10 por 1.000.000 de invocações

Lambda@Edge:
  $0.60 por 1.000.000 de invocações (viewer events)
  + $0.00000625125 por GB-segundo

Exemplo: 100M requests/mês, 1ms avg:
  CloudFront Functions: $10/mês
  Lambda@Edge:          $60/mês + duração ≈ $65/mês
  Diferença:            ~6,5x mais caro
```

---

### 5. Deploy via CDK

[FATO] Lambda@Edge deve ser definida em **us-east-1**. Em uma CDK app multi-stack, isso requer uma stack separada deployada nessa região:

```python
from aws_cdk import (
    App, Stack, Environment,
    aws_lambda as lambda_,
    aws_cloudfront as cloudfront,
    aws_cloudfront_origins as origins,
    aws_s3 as s3,
    Duration,
)

# ── Stack de Edge Functions (deve estar em us-east-1) ──────────────────────────
class EdgeFunctionsStack(Stack):
    def __init__(self, scope, construct_id, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # Lambda@Edge — viewer-request para autenticação JWT
        self.auth_function = cloudfront.experimental.EdgeFunction(
            self, "AuthFunction",
            runtime=lambda_.Runtime.PYTHON_3_12,
            handler="auth.handler",
            code=lambda_.Code.from_asset("src/edge/auth"),
            timeout=Duration.seconds(5),
            memory_size=128,
            description="JWT authentication at edge",
            # Não suporta: environment_variables, layers
        )

# ── Stack principal da aplicação ───────────────────────────────────────────────
class AppStack(Stack):
    def __init__(self, scope, construct_id, edge_stack: EdgeFunctionsStack, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        bucket = s3.Bucket(self, "StaticAssets")

        # CloudFront Function — injetar security headers (nativa, não Lambda)
        security_headers_fn = cloudfront.Function(
            self, "SecurityHeadersFn",
            code=cloudfront.FunctionCode.from_inline("""
function handler(event) {
    var response = event.response;
    var headers = response.headers;
    headers['strict-transport-security'] = { value: 'max-age=31536000; includeSubdomains' };
    headers['x-content-type-options'] = { value: 'nosniff' };
    headers['x-frame-options'] = { value: 'DENY' };
    return response;
}
"""),
            function_name="SecurityHeadersInjection",
            comment="Inject security headers on all responses",
        )

        distribution = cloudfront.Distribution(
            self, "Distribution",
            default_behavior=cloudfront.BehaviorOptions(
                origin=origins.S3BucketOrigin.with_origin_access_control(bucket),
                viewer_protocol_policy=cloudfront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
                cache_policy=cloudfront.CachePolicy.CACHING_OPTIMIZED,
                # Lambda@Edge: autenticação JWT em viewer-request
                edge_lambdas=[
                    cloudfront.EdgeLambda(
                        function_version=edge_stack.auth_function.current_version,
                        event_type=cloudfront.LambdaEdgeEventType.VIEWER_REQUEST,
                        include_body=False,
                    )
                ],
                # CloudFront Function: security headers em viewer-response
                function_associations=[
                    cloudfront.FunctionAssociation(
                        function=security_headers_fn,
                        event_type=cloudfront.FunctionEventType.VIEWER_RESPONSE,
                    )
                ],
            ),
        )

# ── App com múltiplas stacks ───────────────────────────────────────────────────
app = App()

edge_stack = EdgeFunctionsStack(
    app, "EdgeFunctionsStack",
    env=Environment(region="us-east-1")   # OBRIGATÓRIO para Lambda@Edge
)

app_stack = AppStack(
    app, "AppStack",
    edge_stack=edge_stack,
    env=Environment(region="sa-east-1")   # Região principal da aplicação
)

app_stack.add_dependency(edge_stack)
app.synth()
```

[FATO] `cloudfront.experimental.EdgeFunction` no CDK cria automaticamente a função em us-east-1 (independente da região da stack onde é usado), replica para os edge locations, e configura o trust policy correto com `edgelambda.amazonaws.com`.

---

## Exemplo prático

**Cenário:** Plataforma de conteúdo com três requisitos: (a) todos os responses devem ter security headers, (b) usuários autenticados veem conteúdo personalizado — o origin precisa saber o `user_id`, (c) arquivos de imagem devem ser servidos com cache longo sem que parâmetros de analytics poluam o cache.

**Solução arquitetada:**

```
viewer-request (CloudFront Function):
  → Remove query params de tracking (utm_*, fbclid)
  → Normaliza cache key

viewer-request (Lambda@Edge — somente paths /api/*):
  → Verifica JWT
  → Injeta X-User-Id no request para a origem

viewer-response (CloudFront Function):
  → Injeta security headers em todos os responses
```

```javascript
// cloudfront-function-normalize.js (viewer-request)
// Aplica ao behavior padrão (assets estáticos)
function handler(event) {
    var request = event.request;
    var qs = request.querystring;

    // Remove tracking params que não afetam o conteúdo
    ['utm_source','utm_medium','utm_campaign','utm_content',
     'utm_term','fbclid','gclid','_ga'].forEach(function(p) {
        delete qs[p];
    });

    // Normaliza Accept-Language para pt ou en (reduz variações de cache)
    var lang = request.headers['accept-language'];
    if (lang) {
        var value = lang.value || '';
        request.headers['cf-accept-language'] = {
            value: value.startsWith('pt') ? 'pt' : 'en'
        };
    }

    return request;
}
```

```python
# lambda_edge_auth.py (viewer-request — behavior /api/*)
import json, os, base64, hmac, hashlib, boto3

_SECRET = None

def _load_secret():
    global _SECRET
    if _SECRET is None:
        ssm = boto3.client('ssm', region_name='us-east-1')
        _SECRET = ssm.get_parameter(
            Name='/plataforma/jwt-secret', WithDecryption=True
        )['Parameter']['Value']
    return _SECRET

def handler(event, context):
    request = event['Records'][0]['cf']['request']
    headers = request.get('headers', {})
    auth = headers.get('authorization', [{}])[0].get('value', '')

    if not auth.startswith('Bearer '):
        return _deny()

    try:
        payload = _decode_jwt(auth[7:], _load_secret())
        request['headers']['x-user-id'] = [{'key':'X-User-Id','value': payload['sub']}]
        request['headers']['x-user-plan'] = [{'key':'X-User-Plan','value': payload.get('plan','free')}]
        return request
    except Exception:
        return _deny()

def _decode_jwt(token, secret):
    h, p, s = token.split('.')
    sig = base64.urlsafe_b64decode(s + '==')
    expected = hmac.new(secret.encode(), f"{h}.{p}".encode(), hashlib.sha256).digest()
    if not hmac.compare_digest(sig, expected):
        raise ValueError('bad sig')
    return json.loads(base64.urlsafe_b64decode(p + '=='))

def _deny():
    return {
        'status': '401',
        'statusDescription': 'Unauthorized',
        'headers': {'content-type': [{'key':'Content-Type','value':'application/json'}]},
        'body': '{"error":"Unauthorized"}'
    }
```

---

## Armadilhas comuns

### Armadilha 1 — Usar variáveis de ambiente em Lambda@Edge

**O erro:** O desenvolvedor cria a função Lambda@Edge com `environment_variables={"JWT_SECRET": "..."}` no CDK. O deploy falha com erro: `Lambda@Edge does not support environment variables`.

**Por que acontece:** Lambda@Edge replica a função para dezenas de edge locations globais. Variáveis de ambiente são regionais — não há mecanismo para replicá-las junto com o código para cada PoP.

**Como evitar:** Três alternativas em ordem de preferência:
1. **SSM Parameter Store / Secrets Manager** (lido no init phase, cacheado em variável global)
2. **Hardcode** para valores não-sensíveis e estáticos
3. **CloudFront Key Value Store** — um KV store gerenciado da AWS disponível em CloudFront Functions (não Lambda@Edge) para pares chave-valor leves

Para Lambda@Edge, a opção 1 é a padrão de produção: a primeira invocação (cold start) busca o segredo, as subsequentes (warm) reusam o valor cacheado em memória. O custo extra é uma chamada SSM por cold start.

---

### Armadilha 2 — Deploy em região diferente de us-east-1 para Lambda@Edge

**O erro:** O desenvolvedor cria a função Lambda em `sa-east-1` (São Paulo) porque é a região principal da aplicação, depois tenta associá-la ao CloudFront. O erro é: `InvalidLambdaFunctionAssociation: The function ARN must be in the same account as the distribution and must be in us-east-1`.

**Por que acontece:** O sistema de replicação do Lambda@Edge é gerenciado a partir de us-east-1. A AWS precisa de uma função "mestre" em us-east-1 para replicar para os edge locations globais.

**Como evitar:**
- Use `cloudfront.experimental.EdgeFunction` no CDK — ele garante o deploy em us-east-1 automaticamente, independente da região da stack.
- Se usar Lambda diretamente, crie a função manualmente em us-east-1 e referencie o ARN com a versão publicada (não `$LATEST`).
- Nunca use `$LATEST` em Lambda@Edge — apenas versões publicadas são suportadas.

---

### Armadilha 3 — Esquecer que CloudFront Functions é JavaScript síncrono (não Node.js)

**O erro:** O desenvolvedor escreve código com `async/await`, `fetch()`, `require()`, ou acessa `process.env`. O código falha com erros como `ReferenceError: fetch is not defined` ou `SyntaxError: Unexpected token`.

**Por que acontece:** CloudFront Functions usa uma engine JavaScript própria (não Node.js). Não há `process`, não há `require`, não há I/O assíncrono. O runtime é intencionalmente minimalista para garantir execução sub-milissegundo.

**Como evitar:**
- Código síncrono apenas — sem `async`, sem Promises, sem callbacks de I/O
- Sem `require()` — todo o código deve estar em um único arquivo
- Sem `fetch` ou qualquer chamada de rede — se precisar de dados externos, use Lambda@Edge
- Teste sempre no console CloudFront antes de associar à distribuição — ele detecta esses erros antes do deploy
- Para configurações dinâmicas, use **CloudFront Key Value Store** (API do runtime: `CloudFrontFunction.cf.kvs.get()`)

---

## Exercício de reflexão

Você está redesenhando a camada de edge de uma plataforma de streaming de vídeo. Os requisitos são: (1) bloquear requisições de IPs em uma lista de bloqueio que é atualizada diariamente, (2) adicionar um header `X-Country` baseado na geolocalização do CloudFront, (3) verificar se o usuário tem um plano ativo consultando o DynamoDB antes de servir vídeos Premium, e (4) adicionar headers de cache corretos (`Cache-Control`) em todas as respostas.

**Questão:** Para cada um dos quatro requisitos, você usaria CloudFront Functions ou Lambda@Edge? Em qual evento (viewer-request, origin-request, etc.)? Justifique sua escolha considerando latência, custo, acesso a dados externos e impacto no cache. Em particular, para o requisito 3, como você evitaria que a verificação no DynamoDB aconteça a cada request de um usuário já autenticado?

---

## Recursos para aprofundar

1. **Differences between CloudFront Functions and Lambda@Edge**
   URL: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/edge-functions-choosing.html
   A tabela comparativa oficial entre os dois serviços, com todos os limites quantitativos (timeout, memória, package size, eventos, body access) e recomendações de caso de uso. É a primeira página a consultar quando precisar decidir entre os dois.

2. **Customize at the edge with CloudFront Functions**
   URL: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cloudfront-functions.html
   Documentação completa de CloudFront Functions: estrutura do evento, runtime JavaScript, como criar e testar no console, Key Value Store, e galeria de exemplos de uso (redirect, header manipulation, A/B testing). Inclui o tutorial passo-a-passo para criar a primeira função.

3. **Customize at the edge with Lambda@Edge**
   URL: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/lambda-at-the-edge.html
   Guia completo de Lambda@Edge: os quatro event types, estrutura dos eventos de request e response, como configurar triggers, restrições específicas (us-east-1, sem env vars, sem layers), e exemplos avançados (autenticação, geração de responses, manipulação de body).

---

