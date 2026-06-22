# Session 026 — ALB: complex listener rules, weighted routing and fixed responses

**Estimated duration:** 60 minutes
**Prerequisites:** session-014-ecs-services-discovery-alb

---

## Objective

By the end, you will be able to create listener rules with multiple conditions (path + header + query string), configure weighted target groups for canary releases at the ALB level (without CodeDeploy), and add fixed response for external health checks that should not reach the backend.

---

## Context

[FACT] The Application Load Balancer (ALB) operates at layer 7 (HTTP/HTTPS) and routes requests to target groups based on rules defined in the listener. A rule is composed of: numeric priority, zero or more conditions, and exactly one routing action (forward, redirect, or fixed-response).

[CONSENSUS] The expressiveness of ALB listener rules is sufficient to implement sophisticated patterns that on other platforms would require a separate reverse proxy: canary deployments by traffic weight, synthetic responses for external probes, routing by API version via header, and blocking paths without touching the backend. Understanding this mechanism reduces the need for additional layers (Nginx, Envoy) for common cases.

---

## Core concepts

### 1. Anatomy of a listener rule

Every rule in the ALB has three parts:

```
┌─────────────────────────────────────────────────────────────┐
│                     LISTENER (port 443)                      │
├────────┬─────────────────────────────┬──────────────────────┤
│Priority│        Conditions           │       Action          │
├────────┼─────────────────────────────┼──────────────────────┤
│   10   │ path=/api/v2/*              │ → forward TG-v2      │
│        │ AND header[X-Beta]=true     │                      │
├────────┼─────────────────────────────┼──────────────────────┤
│   20   │ path=/health                │ fixed-response 200   │
├────────┼─────────────────────────────┼──────────────────────┤
│   30   │ path=/api/*                 │ → forward TG-v1 (90%)│
│        │                             │             TG-v2(10%)│
├────────┼─────────────────────────────┼──────────────────────┤
│ 65535  │ (default — no conditions)   │ → forward TG-v1      │
└────────┴─────────────────────────────┴──────────────────────┘
```

**Priority:** integer value from 1 to 50,000. Rules evaluated in ascending order. The first rule whose conditions are satisfied executes its action and evaluation stops. The default rule (priority 65535) has no conditions and is created automatically when creating the listener — it cannot be removed, only have its action changed.

**AND/OR semantics:** multiple conditions within a rule are combined with AND (all must be true). Within a single condition, multiple values are combined with OR (any value suffices).

```
Rule: path=/api/v2/*  AND  http-header[X-Env]=staging
                            ↑
                     Both must be TRUE

Condition: path-pattern Values=["/api/v1/*", "/api/v2/*"]
                                 ↑
                        Any one suffices (OR)
```

[FACT] Limits per rule (per official documentation):
- Maximum of 5 *match evaluations* (comparisons) per rule
- Maximum of 3 comparison strings per condition
- Maximum of 5 wildcards (`*` and `?`) per rule
- Exactly zero or one of: `host-header`, `http-request-method`, `path-pattern`, `source-ip`
- Zero or more of: `http-header`, `query-string`

---

### 2. Condition types in detail

ALB supports six condition types. The table below summarizes the behavior and particularities of each:

| Type              | Case-sensitive | Wildcard | Regex | Multiple per rule |
|-------------------|:--------------:|:--------:|:-----:|:-----------------:|
| `host-header`     | No             | Yes      | Yes   | No                |
| `http-header`     | No (value)     | Yes      | Yes   | Yes               |
| `http-request-method` | Yes        | No       | No    | No                |
| `path-pattern`    | **Yes**        | Yes      | Yes   | No                |
| `query-string`    | No             | Yes      | No    | Yes               |
| `source-ip`       | N/A (CIDR)     | No       | No    | No                |

**path-pattern** is case-sensitive — `/API/v1/*` and `/api/v1/*` are distinct patterns. This is a frequent pitfall in API migrations that previously lived behind a case-insensitive Nginx.

**http-header** uses the header name case-insensitively, and the value is also case-insensitive. You can create multiple `http-header` conditions in the same rule to require more than one header simultaneously (via AND).

**query-string** accepts key/value pairs or value only. A `null` key means "any key with this value":

```json
// Condition: satisfied if query has version=v2 OR any key=beta
{
  "Field": "query-string",
  "QueryStringConfig": {
    "Values": [
      { "Key": "version", "Value": "v2" },
      { "Value": "beta" }
    ]
  }
}
```

**source-ip** evaluates the TCP connection IP, not the `X-Forwarded-For`. If the client is behind a proxy, the evaluated IP is the proxy's. To evaluate the original client IP, use an `http-header` condition on the `X-Forwarded-For` field.

---

### 3. Weighted target groups for canary releases

The ALB allows a single `forward` action to distribute traffic among multiple target groups with independent weights. Each weight is an integer from 0 to 999. The traffic proportion for each TG is `TG_weight / total_weight_sum`.

```
Example: TG-v1 (weight=90) + TG-v2 (weight=10)
  → TG-v1 receives 90/(90+10) = 90% of requests
  → TG-v2 receives 10/(90+10) = 10% of requests

Progressive canary (updating weights via API):
  Phase 1: v1=95, v2=5    →  5% on canary
  Phase 2: v1=80, v2=20   → 20% on canary
  Phase 3: v1=50, v2=50   → 50% on canary
  Phase 4: v1=0,  v2=100  → full rollout
  (TG-v1 with weight=0: rule exists but TG receives no traffic)
```

[FACT] Balancing between TGs is done at the rule level, not per session — each request is distributed independently based on weights. To ensure a specific user is always sent to the same TG during a session, enable **target group stickiness** on the rule. The ALB then emits a cookie `AWSALBTG` that encodes the selected TG; subsequent requests with this cookie ignore weight distribution and go directly to the indicated TG.

[FACT] Behavior with empty or unhealthy TG: if a TG configured with weight > 0 has no healthy targets, the ALB does **not** automatically failover to the other TG. Requests routed to the TG without targets return 502 or 503. This is different from single TG behavior, where the ALB simply has nowhere to route. In canary environments, actively monitor the canary TG before increasing weight.

```
Canary diagram with sticky sessions:

  Request 1 (new user)
       │
       ▼
  ALB Rule: TG-v1(80) + TG-v2(20)
       │
       ├── 80% → TG-v1 → [Set-Cookie: AWSALBTG=<hash-v1>]
       └── 20% → TG-v2 → [Set-Cookie: AWSALBTG=<hash-v2>]

  Request 2 (same user, with cookie AWSALBTG=<hash-v1>)
       │
       ▼
  ALB Rule: detects cookie → ignores weights → goes directly to TG-v1
```

---

### 4. Fixed-response action

The `fixed-response` action makes the ALB return a synthetic HTTP response without forwarding the request to any target. The payload is configured directly in the rule:

```json
{
  "Type": "fixed-response",
  "FixedResponseConfig": {
    "StatusCode": "200",
    "ContentType": "text/plain",
    "MessageBody": "OK"
  }
}
```

[FACT] `StatusCode` can be any 2XX, 4XX, or 5XX code. `ContentType` accepts: `text/plain`, `text/css`, `text/html`, `application/javascript`, `application/json`. `MessageBody` is optional; maximum 1024 bytes.

Main use cases:

1. **Health check for uptime tools**: services like Pingdom, Datadog Synthetics, or Route 53 Health Checks can verify `/health` without that request reaching the backend. Reduces load and eliminates the need for a health check endpoint on application servers.

2. **Path blocking**: return 403 for legacy routes that should no longer exist, without keeping code or logic in the backend.

3. **Planned maintenance**: during a disruptive deploy, a temporary high-priority rule captures all traffic and returns 503 with a custom message.

4. **CORS preflight in layers without backend**: return 204 for `OPTIONS` on static paths without forwarding to containers.

[FACT] When a `fixed-response` action is executed, it's recorded in ALB access logs and counted in the `HTTP_Fixed_Response_Count` CloudWatch metric.

---

## Practical example

**Scenario:** an e-commerce service with a versioned API needs:
1. Route `/api/v2/*` only when header `X-Api-Version: 2` is present → TG-v2
2. Implement 10% canary for general `/api/*`, directing 10% to TG-v2
3. Respond `200 OK` for `/health` without reaching the backend
4. Send everything else to TG-v1 (default)

### CDK Python

```python
from aws_cdk import Stack, aws_elasticloadbalancingv2 as elbv2
import aws_cdk.aws_elasticloadbalancingv2 as elbv2
from constructs import Construct


class AlbRulesStack(Stack):
    def __init__(self, scope: Construct, id: str, **kwargs):
        super().__init__(scope, id, **kwargs)

        # ALB, VPC, TGs created previously (referenced by ARN or import)
        alb = elbv2.ApplicationLoadBalancer.from_lookup(
            self, "ALB",
            load_balancer_arn="arn:aws:elasticloadbalancing:...",
        )

        tg_v1 = elbv2.ApplicationTargetGroup.from_target_group_attributes(
            self, "TG-v1",
            target_group_arn="arn:aws:elasticloadbalancing:...:targetgroup/api-v1/...",
        )
        tg_v2 = elbv2.ApplicationTargetGroup.from_target_group_attributes(
            self, "TG-v2",
            target_group_arn="arn:aws:elasticloadbalancing:...:targetgroup/api-v2/...",
        )

        listener = elbv2.ApplicationListener.from_lookup(
            self, "Listener",
            load_balancer_arn=alb.load_balancer_arn,
            listener_port=443,
        )

        # Rule 1 (priority 10): /api/v2/* + header X-Api-Version: 2 → TG-v2 (100%)
        listener.add_action(
            "RouteV2WithHeader",
            priority=10,
            conditions=[
                elbv2.ListenerCondition.path_patterns(["/api/v2/*"]),
                elbv2.ListenerCondition.http_header(
                    "X-Api-Version", ["2"]
                ),
            ],
            action=elbv2.ListenerAction.forward([tg_v2]),
        )

        # Rule 2 (priority 20): /health → fixed-response 200
        listener.add_action(
            "HealthCheck",
            priority=20,
            conditions=[
                elbv2.ListenerCondition.path_patterns(["/health"]),
            ],
            action=elbv2.ListenerAction.fixed_response(
                status_code=200,
                content_type=elbv2.ContentType.TEXT_PLAIN,
                message_body="OK",
            ),
        )

        # Rule 3 (priority 30): /api/* → 10% canary to TG-v2, 90% to TG-v1
        # With target group stickiness of 1 hour
        listener.add_action(
            "CanaryApi",
            priority=30,
            conditions=[
                elbv2.ListenerCondition.path_patterns(["/api/*"]),
            ],
            action=elbv2.ListenerAction.weighted_forward(
                target_groups=[
                    elbv2.WeightedTargetGroup(target_group=tg_v1, weight=90),
                    elbv2.WeightedTargetGroup(target_group=tg_v2, weight=10),
                ],
                stickiness_duration=Duration.hours(1),
            ),
        )

        # Default rule (created when creating the listener): → TG-v1
        # Added via listener default_action at creation time:
        #   elbv2.ApplicationListener(..., default_action=ListenerAction.forward([tg_v1]))
```

### CLI — creating and modifying weights at runtime

```bash
# Create canary rule (10% v2)
aws elbv2 create-rule \
  --listener-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:listener/app/my-alb/... \
  --priority 30 \
  --conditions '[{"Field":"path-pattern","PathPatternConfig":{"Values":["/api/*"]}}]' \
  --actions '[{
    "Type": "forward",
    "ForwardConfig": {
      "TargetGroups": [
        {"TargetGroupArn": "arn:...targetgroup/api-v1/...", "Weight": 90},
        {"TargetGroupArn": "arn:...targetgroup/api-v2/...", "Weight": 10}
      ],
      "TargetGroupStickinessConfig": {"Enabled": true, "DurationSeconds": 3600}
    }
  }]'

# Scale canary to 50%: modify weights without recreating the rule
aws elbv2 modify-rule \
  --rule-arn arn:aws:elasticloadbalancing:...:rule/... \
  --actions '[{
    "Type": "forward",
    "ForwardConfig": {
      "TargetGroups": [
        {"TargetGroupArn": "arn:...targetgroup/api-v1/...", "Weight": 50},
        {"TargetGroupArn": "arn:...targetgroup/api-v2/...", "Weight": 50}
      ],
      "TargetGroupStickinessConfig": {"Enabled": true, "DurationSeconds": 3600}
    }
  }]'

# Full rollout: weight 0 on v1
aws elbv2 modify-rule \
  --rule-arn arn:aws:elasticloadbalancing:...:rule/... \
  --actions '[{
    "Type": "forward",
    "ForwardConfig": {
      "TargetGroups": [
        {"TargetGroupArn": "arn:...targetgroup/api-v1/...", "Weight": 0},
        {"TargetGroupArn": "arn:...targetgroup/api-v2/...", "Weight": 100}
      ]
    }
  }]'

# List all listener rules
aws elbv2 describe-rules \
  --listener-arn arn:aws:elasticloadbalancing:...:listener/... \
  --query 'Rules[*].{Priority:Priority,Conditions:Conditions[*].Field,Action:Actions[0].Type}' \
  --output table
```

---

## Common pitfalls

### 1. Multiple conditions as OR instead of AND
**The mistake:** a developer wants to route requests that HAVE `/api/v2/*` AND header `X-Beta: true`. They create two separate rules, one per condition, both pointing to TG-v2. Result: any request to `/api/v2/*` (without the header) goes to TG-v2, and any request with `X-Beta: true` (any path) also goes.

**Why it happens:** the developer thinks of each condition as an additional filter. In reality, separate rules are evaluated independently — if any one of them matches, it executes.

**How to avoid:** conditions that must be combined with AND need to be in the same rule. A rule with `path-pattern=/api/v2/*` AND `http-header[X-Beta]=true` only matches if both are simultaneously true.

---

### 2. TG with weight 0 vs removing the TG from the rule
**The mistake:** to "pause" the canary, the team changes TG-v2's weight to 0. After weeks, raises it back to 10% without testing the TG's state.

**Why it happens:** weight 0 does not mean the TG is removed from the balancing — it still exists in the rule and the ALB will monitor its targets. If TG-v2 only has deregistered targets and stickiness is active, users who already received the `AWSALBTG` cookie for v2 will continue being routed to it, resulting in 502.

**How to avoid:** when doing a complete rollback, remove the TG from the rule and use simple `forward` to a single TG. Use weight 0 only as a brief transitional step. Always verify TG health before increasing weight again.

---

### 3. path-pattern is case-sensitive
**The mistake:** the application accepts `/API/users` and `/api/users` indiscriminately (standard in some frameworks), but the rule uses `/api/*`. Requests to `/API/*` end up at the default rule, often resulting in unexpected behavior.

**Why it happens:** path-pattern in ALB is case-sensitive, per documentation. Many developers assume HTTP behavior (where the path may or may not be case-sensitive depending on the server).

**How to avoid:** normalize URLs in the application or CDN before they reach the ALB. If necessary, create duplicate rules for case variants. Alternatively, use regex matching (`RegexValues`) with case-insensitive flag: `^(?i)/api/.*$`.

---

## Reflection exercise

You are responsible for a REST API with two critical endpoints: `/api/payments/*` and `/api/catalog/*`. The team decided to migrate payments to a new service version (v2) gradually, but wants the migration to be controllable per client: enterprise clients (identified by header `X-Client-Tier: enterprise`) should go directly to v2 from the start, while the rest goes through normal 5% canary to v2.

Additionally, Datadog external monitoring checks `/health/ping` every 30 seconds — you want this request to return 200 without reaching the backend.

Describe the complete structure of the necessary listener rules: which priorities to assign, which conditions to combine in each rule, which actions to configure, and what behaviors would emerge if the priority order were swapped between some rules. Also consider what happens if the payments TG-v2 becomes unhealthy during the canary.

---

## Resources for further study

**[Listener rules for your Application Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/listener-rules.html)**
Main documentation: rule structure, how to create/modify/reorder. Section of interest: "Rule evaluation order" and "Default rules".

**[Condition types for listener rules](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/rule-condition-types.html)**
Complete reference for each condition type with CLI JSON examples. Includes regex matching examples for `host-header` and `path-pattern`.

**[Action types for listener rules](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/rule-action-types.html)**
Covers forward (simple and weighted), fixed-response, and redirect with all fields. Section "Sticky sessions and weighted target groups" explains the `AWSALBTG` cookie.

---
