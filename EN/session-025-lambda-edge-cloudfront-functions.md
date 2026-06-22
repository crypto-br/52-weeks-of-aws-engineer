# Session 025 — Lambda@Edge vs CloudFront Functions: use cases and limits

**Estimated duration:** 60 minutes
**Prerequisites:** session-024-lambda-observabilidade-xray-insights

---

## Objective

By the end, you will be able to choose between Lambda@Edge and CloudFront Functions for a given requirement (based on execution latency, body access, number of regions, cost, and deploy time), implement a simple header injection with CloudFront Functions, and understand the timeout and memory limits of each option.

---

## Context

[FACT] CloudFront is the AWS CDN (Content Delivery Network). It distributes content from more than 600 points of presence (PoPs) around the world, called **edge locations**. When a request arrives at CloudFront, it's processed at the edge location closest to the user — not in the AWS region of the origin. This means any logic executed at the edge has radically lower latency than a call back to the origin.

[FACT] Two services allow executing code at the CloudFront edge: **CloudFront Functions** (launched in 2021) and **Lambda@Edge** (launched in 2017). They differ fundamentally in capability, latency, cost, and use cases. They are not direct competitors — each solves a different class of problems.

[CONSENSUS] The general rule adopted by most production architectures: if the logic is simple (header manipulation, URL rewrites, cache key normalization) and runs on each request, use CloudFront Functions. If the logic is complex (body access, calls to external services, complete JWT authentication, decisions based on databases), use Lambda@Edge. The cost of CloudFront Functions is approximately 1/6 of Lambda@Edge per request.

---

## Core concepts

### 1. The four CloudFront interception points

[FACT] CloudFront intercepts the HTTP flow at four distinct points. Each point has different characteristics and supports different types of edge functions:

```
User
   │
   ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                         CLOUDFRONT EDGE LOCATION                            │
│                                                                              │
│  ① viewer-request                              ② viewer-response            │
│  Before cache check                            Before returning to user      │
│  (CloudFront Functions ✓)                      (CloudFront Functions ✓)      │
│  (Lambda@Edge ✓)                               (Lambda@Edge ✓)              │
│         │                                              ▲                    │
│         ▼                                              │                    │
│   ┌──────────────┐                           ┌─────────────────┐           │
│   │  Cache hit?  │──── HIT ─────────────────►│ Serve from cache│           │
│   └──────┬───────┘                           └─────────────────┘           │
│          │ MISS                                                              │
│          ▼                                                                   │
│  ③ origin-request            ④ origin-response                             │
│  Before going to origin      After receiving from origin (and caching)       │
│  (Lambda@Edge ✓ only)        (Lambda@Edge ✓ only)                           │
│         │                                    ▲                              │
└─────────┼────────────────────────────────────┼──────────────────────────────┘
          │                                    │
          ▼                                    │
       ORIGIN (S3, ALB, EC2, API Gateway...)───┘
```

[FACT] **CloudFront Functions** can only be associated with `viewer-request` and `viewer-response`. **Lambda@Edge** can be associated with all four events. This distinction is fundamental: only Lambda@Edge can intercept and modify requests/responses to the origin.

---

### 2. CloudFront Functions — sub-millisecond, massive scale, strict limitations

[FACT] CloudFront Functions executes JavaScript (ES2015+) code in a **proprietary** runtime — it's not Node.js. It's a lightweight JavaScript engine, without npm modules, without network access, without asynchronous timers, without `require()`. The code must be **synchronous** and finish in less than 1ms of compute time.

#### Limits and capabilities

[FACT] Documented limits:

```
┌──────────────────────────────────┬──────────────────────────────────────────┐
│ Limit                            │ Value                                    │
├──────────────────────────────────┼──────────────────────────────────────────┤
│ Maximum execution time           │ ~1ms (compute utilization 0-100)        │
│ Memory                           │ 2 MB                                     │
│ Maximum code size                │ 10 KB                                    │
│ Runtime                          │ JavaScript (ES2015+) — NOT Node.js      │
│ Request body access              │ NO                                       │
│ Network calls                    │ NO                                       │
│ Environment variables            │ NO (use Key Value Store)                 │
│ Filesystem access                │ NO                                       │
│ Supported events                 │ viewer-request, viewer-response          │
│ Scale                            │ Millions of req/s instantly              │
│ Deploy                           │ CloudFront native (not via Lambda)       │
│ Integrated testing               │ CloudFront console                       │
└──────────────────────────────────┴──────────────────────────────────────────┘
```

#### Event structure (CloudFront Functions)

[FACT] The event object has a different structure from Lambda@Edge:

```javascript
// event received by the function (viewer-request)
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

[FACT] The function must return the `request` object (for viewer-request) or `response` object (for viewer-response). If it returns a `response` object in viewer-request, the request is **interrupted** — it never reaches the cache or origin.

#### CloudFront Functions examples

```javascript
// 1. Inject Security Headers (viewer-response)
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
// 2. Normalize cache key — remove irrelevant query strings (viewer-request)
// Without this, "?utm_source=email" and "?utm_source=social" generate separate cache misses
function handler(event) {
    var request = event.request;
    var qs = request.querystring;

    // Remove tracking parameters — they don't affect content
    var tracking = ['utm_source', 'utm_medium', 'utm_campaign', 'utm_content', 'fbclid', 'gclid'];
    tracking.forEach(function(param) { delete qs[param]; });

    return request;
}
```

```javascript
// 3. URL rewrite — SPA with all routes serving index.html (viewer-request)
function handler(event) {
    var request = event.request;
    var uri = request.uri;

    // If it has no file extension, serve index.html
    if (!uri.includes('.')) {
        request.uri = '/index.html';
    }
    return request;
}
```

```javascript
// 4. Redirect HTTP → HTTPS and www → apex (viewer-request)
function handler(event) {
    var request = event.request;
    var headers = request.headers;
    var host = headers.host ? headers.host.value : '';

    // Redirect www to apex
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
// Assigns new users to a variant and redirects to the correct path
function handler(event) {
    var request = event.request;
    var cookies = request.cookies;

    // User already has assigned variant
    if (cookies['ab-variant']) {
        var variant = cookies['ab-variant'].value;
        request.uri = '/' + variant + request.uri;
        return request;
    }

    // Assign new variant (50/50)
    var variant = Math.random() < 0.5 ? 'a' : 'b';
    request.uri = '/' + variant + request.uri;

    // Return response to set cookie — next request won't be assigned
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

### 3. Lambda@Edge — full power, greater operational complexity

[FACT] Lambda@Edge are regular Lambda functions, with the following specific restrictions for edge execution:

```
┌─────────────────────────────────┬──────────────────────┬──────────────────────┐
│ Limit                           │ Viewer events        │ Origin events         │
├─────────────────────────────────┼──────────────────────┼──────────────────────┤
│ Maximum timeout                 │ 5 seconds            │ 30 seconds            │
├─────────────────────────────────┼──────────────────────┼──────────────────────┤
│ Maximum memory                  │ 128 MB               │ 128 MB – 10,240 MB    │
├─────────────────────────────────┼──────────────────────┼──────────────────────┤
│ Package size (compressed)       │ 1 MB                 │ 50 MB                 │
├─────────────────────────────────┼──────────────────────┼──────────────────────┤
│ Body access                     │ Yes (40 KB)          │ Yes (1 MB)            │
├─────────────────────────────────┼──────────────────────┼──────────────────────┤
│ Network calls                   │ Yes                  │ Yes                   │
├─────────────────────────────────┼──────────────────────┼──────────────────────┤
│ Environment variables           │ NO                   │ NO                    │
├─────────────────────────────────┼──────────────────────┼──────────────────────┤
│ Lambda Layers                   │ NO                   │ NO                    │
├─────────────────────────────────┼──────────────────────┼──────────────────────┤
│ Runtime                         │ Node.js, Python      │ Node.js, Python       │
├─────────────────────────────────┼──────────────────────┼──────────────────────┤
│ Deploy                          │ us-east-1 only       │ us-east-1 only        │
└─────────────────────────────────┴──────────────────────┴──────────────────────┘
```

[FACT] Lambda@Edge **must be deployed in the us-east-1 region**. Replication to all edge locations worldwide is done automatically by CloudFront when associating the function to a distribution. You manage only the function in us-east-1.

[FACT] Lambda@Edge **does not support environment variables**. Configurations that would normally go in environment variables must be hardcoded, fetched from SSM Parameter Store/Secrets Manager at runtime (with cache in the init phase), or injected via CloudFormation during deploy.

[FACT] The **execution role** of the Lambda@Edge function must have both `lambda.amazonaws.com` and `edgelambda.amazonaws.com` as trusted principals:

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

---

### 4. Decision table — CloudFront Functions vs Lambda@Edge

[FACT] The official AWS documentation summarizes the decision as:

```
USE CloudFront Functions when:             USE Lambda@Edge when:
──────────────────────────────────────────   ──────────────────────────────────────────
✓ URL normalization and rewrites             ✓ Complex authentication/authorization
✓ Header manipulation (add/remove/mod)       ✓ Logic depending on request body
✓ Normalize cache keys (remove params)       ✓ Calls to AWS (DynamoDB, SSM, etc.)
✓ Simple cookie manipulation                 ✓ Different response by geolocation
✓ A/B redirect by cookie                    ✓ Image processing
✓ Redirect HTTP → HTTPS                     ✓ Dynamic response generation
✓ Inject security headers                   ✓ URL rewrite based on external data
✓ Minimal cost (millions of req/s)          ✓ Intercept origin-request/response
✓ Instant deploy                            ✓ Modify request before going to S3/ALB
```

**Comparative cost (approximate):**

```
CloudFront Functions:
  $0.10 per 1,000,000 invocations

Lambda@Edge:
  $0.60 per 1,000,000 invocations (viewer events)
  + $0.00000625125 per GB-second

Example: 100M requests/month, 1ms avg:
  CloudFront Functions: $10/month
  Lambda@Edge:          $60/month + duration ≈ $65/month
  Difference:           ~6.5x more expensive
```

---

### 5. Deploy via CDK

[FACT] Lambda@Edge must be defined in **us-east-1**. In a multi-stack CDK app, this requires a separate stack deployed in that region:

```python
from aws_cdk import (
    App, Stack, Environment,
    aws_lambda as lambda_,
    aws_cloudfront as cloudfront,
    aws_cloudfront_origins as origins,
    aws_s3 as s3,
    Duration,
)

# ── Edge Functions Stack (must be in us-east-1) ──────────────────────────
class EdgeFunctionsStack(Stack):
    def __init__(self, scope, construct_id, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # Lambda@Edge — viewer-request for JWT authentication
        self.auth_function = cloudfront.experimental.EdgeFunction(
            self, "AuthFunction",
            runtime=lambda_.Runtime.PYTHON_3_12,
            handler="auth.handler",
            code=lambda_.Code.from_asset("src/edge/auth"),
            timeout=Duration.seconds(5),
            memory_size=128,
            description="JWT authentication at edge",
            # Not supported: environment_variables, layers
        )

# ── Main application stack ───────────────────────────────────────────────
class AppStack(Stack):
    def __init__(self, scope, construct_id, edge_stack: EdgeFunctionsStack, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        bucket = s3.Bucket(self, "StaticAssets")

        # CloudFront Function — inject security headers (native, not Lambda)
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
                # Lambda@Edge: JWT authentication in viewer-request
                edge_lambdas=[
                    cloudfront.EdgeLambda(
                        function_version=edge_stack.auth_function.current_version,
                        event_type=cloudfront.LambdaEdgeEventType.VIEWER_REQUEST,
                        include_body=False,
                    )
                ],
                # CloudFront Function: security headers in viewer-response
                function_associations=[
                    cloudfront.FunctionAssociation(
                        function=security_headers_fn,
                        event_type=cloudfront.FunctionEventType.VIEWER_RESPONSE,
                    )
                ],
            ),
        )

# ── App with multiple stacks ───────────────────────────────────────────────
app = App()

edge_stack = EdgeFunctionsStack(
    app, "EdgeFunctionsStack",
    env=Environment(region="us-east-1")   # REQUIRED for Lambda@Edge
)

app_stack = AppStack(
    app, "AppStack",
    edge_stack=edge_stack,
    env=Environment(region="sa-east-1")   # Main application region
)

app_stack.add_dependency(edge_stack)
app.synth()
```

[FACT] `cloudfront.experimental.EdgeFunction` in CDK automatically creates the function in us-east-1 (regardless of the region of the stack where it's used), replicates to edge locations, and configures the correct trust policy with `edgelambda.amazonaws.com`.

---

## Practical example

**Scenario:** Content platform with three requirements: (a) all responses must have security headers, (b) authenticated users see personalized content — the origin needs to know the `user_id`, (c) image files must be served with long cache without analytics parameters polluting the cache.

**Architected solution:**

```
viewer-request (CloudFront Function):
  → Remove tracking query params (utm_*, fbclid)
  → Normalize cache key

viewer-request (Lambda@Edge — only /api/* paths):
  → Verify JWT
  → Inject X-User-Id in request to origin

viewer-response (CloudFront Function):
  → Inject security headers on all responses
```

---

## Common pitfalls

### Pitfall 1 — Using environment variables in Lambda@Edge

**The mistake:** The developer creates the Lambda@Edge function with `environment_variables={"JWT_SECRET": "..."}` in CDK. The deploy fails with error: `Lambda@Edge does not support environment variables`.

**Why it happens:** Lambda@Edge replicates the function to dozens of global edge locations. Environment variables are regional — there's no mechanism to replicate them along with the code to each PoP.

**How to avoid:** Three alternatives in order of preference:
1. **SSM Parameter Store / Secrets Manager** (read in init phase, cached in global variable)
2. **Hardcode** for non-sensitive and static values
3. **CloudFront Key Value Store** — an AWS managed KV store available in CloudFront Functions (not Lambda@Edge) for lightweight key-value pairs

For Lambda@Edge, option 1 is the production standard: the first invocation (cold start) fetches the secret, subsequent ones (warm) reuse the value cached in memory. The extra cost is one SSM call per cold start.

---

### Pitfall 2 — Deploy in a region other than us-east-1 for Lambda@Edge

**The mistake:** The developer creates the Lambda function in `sa-east-1` (São Paulo) because it's the application's main region, then tries to associate it with CloudFront. The error is: `InvalidLambdaFunctionAssociation: The function ARN must be in the same account as the distribution and must be in us-east-1`.

**Why it happens:** The Lambda@Edge replication system is managed from us-east-1. AWS needs a "master" function in us-east-1 to replicate to global edge locations.

**How to avoid:**
- Use `cloudfront.experimental.EdgeFunction` in CDK — it ensures deploy in us-east-1 automatically, regardless of the stack's region.
- If using Lambda directly, create the function manually in us-east-1 and reference the ARN with the published version (not `$LATEST`).
- Never use `$LATEST` in Lambda@Edge — only published versions are supported.

---

### Pitfall 3 — Forgetting that CloudFront Functions is synchronous JavaScript (not Node.js)

**The mistake:** The developer writes code with `async/await`, `fetch()`, `require()`, or accesses `process.env`. The code fails with errors like `ReferenceError: fetch is not defined` or `SyntaxError: Unexpected token`.

**Why it happens:** CloudFront Functions uses a proprietary JavaScript engine (not Node.js). There's no `process`, no `require`, no asynchronous I/O. The runtime is intentionally minimalist to guarantee sub-millisecond execution.

**How to avoid:**
- Synchronous code only — no `async`, no Promises, no I/O callbacks
- No `require()` — all code must be in a single file
- No `fetch` or any network call — if you need external data, use Lambda@Edge
- Always test in the CloudFront console before associating with the distribution — it detects these errors before deploy
- For dynamic configurations, use **CloudFront Key Value Store** (runtime API: `CloudFrontFunction.cf.kvs.get()`)

---

## Reflection exercise

You are redesigning the edge layer of a video streaming platform. The requirements are: (1) block requests from IPs on a blocklist that is updated daily, (2) add an `X-Country` header based on CloudFront geolocation, (3) verify if the user has an active plan by querying DynamoDB before serving Premium videos, and (4) add correct cache headers (`Cache-Control`) to all responses.

**Question:** For each of the four requirements, would you use CloudFront Functions or Lambda@Edge? At which event (viewer-request, origin-request, etc.)? Justify your choice considering latency, cost, access to external data, and impact on cache. In particular, for requirement 3, how would you avoid having the DynamoDB check happen on every request for an already authenticated user?

---

## Resources for further study

1. **Differences between CloudFront Functions and Lambda@Edge**
   URL: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/edge-functions-choosing.html
   The official comparative table between the two services, with all quantitative limits (timeout, memory, package size, events, body access) and use case recommendations. It's the first page to consult when you need to decide between the two.

2. **Customize at the edge with CloudFront Functions**
   URL: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cloudfront-functions.html
   Complete CloudFront Functions documentation: event structure, JavaScript runtime, how to create and test in the console, Key Value Store, and usage example gallery (redirect, header manipulation, A/B testing). Includes the step-by-step tutorial to create your first function.

3. **Customize at the edge with Lambda@Edge**
   URL: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/lambda-at-the-edge.html
   Complete Lambda@Edge guide: the four event types, request and response event structure, how to configure triggers, specific restrictions (us-east-1, no env vars, no layers), and advanced examples (authentication, response generation, body manipulation).

---
