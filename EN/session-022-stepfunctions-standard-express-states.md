# Session 22 — Step Functions: Standard vs Express, basic states (Task, Choice, Wait) and execution

**Estimated duration:** 60 minutes
**Prerequisites:** session-021-lambda-extensions-layers-powertools

---

## Objective

By the end, you will be able to create a state machine with Task, Choice, Wait, Succeed, and Fail states, execute it via console and CLI, understand the execution guarantees and cost of Standard vs Express Workflows, and read an execution history to identify where an execution failed.

---

## Context

[FACT] AWS Step Functions is a serverless workflow orchestration service launched in 2016. Unlike messaging solutions (SQS, SNS) that only transport data, Step Functions coordinates *sequences of states* — including conditional logic, waits, parallelism, and error handling — in a declarative, durable, and auditable way.

[CONSENSUS] The core value of the service is removing from application code the responsibility of orchestrating long-running or multi-step flows. Without Step Functions, this state is frequently managed via database flags, chained queues, or retry logic embedded in Lambda functions — patterns that are difficult to monitor, test, and debug. Step Functions externalizes this state to the service itself, making the flow visible, reproducible, and auditable in the console.

[FACT] The language used to define state machines is the **Amazon States Language (ASL)**, a JSON format described in an open specification at [states-language.net](https://states-language.net/spec.html). ASL defines eight state types: Task, Choice, Wait, Succeed, Fail, Pass, Parallel, and Map.

---

## Core concepts

### 1. Standard Workflows vs Express Workflows

Two radically different modes of operation exist in Step Functions. The choice between them affects execution guarantees, auditability, maximum duration, and pricing model.

[FACT] The table below summarizes the differences documented by AWS:

```
┌─────────────────────────┬──────────────────────────┬──────────────────────────────────┐
│ Dimension               │ Standard Workflow         │ Express Workflow                  │
├─────────────────────────┼──────────────────────────┼──────────────────────────────────┤
│ Execution guarantee     │ Exactly-once             │ Async: at-least-once             │
│                         │                          │ Sync:  at-most-once              │
├─────────────────────────┼──────────────────────────┼──────────────────────────────────┤
│ Maximum duration        │ 1 year                   │ 5 minutes                         │
├─────────────────────────┼──────────────────────────┼──────────────────────────────────┤
│ Throughput              │ > 2,000 exec/s           │ > 100,000 exec/s                  │
├─────────────────────────┼──────────────────────────┼──────────────────────────────────┤
│ Execution history       │ GetExecutionHistory API  │ CloudWatch Logs (mandatory)       │
├─────────────────────────┼──────────────────────────┼──────────────────────────────────┤
│ Pricing model           │ Per state transition      │ Per execution + GB-second         │
│                         │ $0.025 / 1,000 trans.    │ $1.00 / 1M exec + $0.00001/GB·s  │
├─────────────────────────┼──────────────────────────┼──────────────────────────────────┤
│ Ideal use case          │ Business workflows,       │ High volume, event-driven,       │
│                         │ durable processes         │ IoT, streaming, microservices    │
└─────────────────────────┴──────────────────────────┴──────────────────────────────────┘
```

[FACT] **Exactly-once** in Standard Workflows means each state is initiated at most once, unless you explicitly configure `Retry` on the state. The service guarantees this by persisting state in its backend, even if internal failures occur in AWS infrastructure.

[FACT] **At-least-once** in asynchronous Express Workflows means the execution *may* start more than once in internal failure scenarios. This implies that the called services must be **idempotent** — the same input executed twice cannot produce distinct side effects.

[CONSENSUS] The most common rule of thumb used by the community: if the workflow involves financial transactions, human approvals, or any non-idempotent operation with duration > 5 min → Standard. If it involves high-frequency event processing where idempotency is natural → Express.

**Comparative cost calculation (example):**

Scenario: 10,000 executions/day, 5 states per execution, 30 days.
- Standard: 10,000 × 5 × 30 = 1,500,000 transitions × $0.025/1,000 = **$37.50/month**
- Express (100ms, 64MB): 300,000 exec × $1.00/1M + 300,000 × 0.1s × 64MB/1024 × $0.00001 = $0.30 + **$0.019 ≈ $0.32/month**

[UNCERTAIN] The prices above reflect AWS documentation as of May 2026. Always check the updated pricing page, as Express Workflows have had price adjustments in recent years.

---

### 2. Amazon States Language (ASL) — state machine structure

[FACT] An ASL definition is a JSON object with the following required fields:

```json
{
  "Comment": "Optional description",
  "StartAt": "FirstStateName",
  "States": {
    "FirstStateName": { ... },
    "AnotherState": { ... }
  }
}
```

`StartAt` points to the name of a state within `States`. Each state must have a `Type` field and, for non-terminal states, a `Next` field (or `End: true` for the final state).

```
Initial state                 Transition                    Terminal state
──────────────────────────────────────────────────────────────────────────────
{                                                          {
  "Type": "Task",       ──── "Next": "NextState" ────       "Type": "Succeed"
  "Resource": "...",                                        (or "Fail")
  "Next": "NextState"                                     }
}
```

[FACT] Transition rules:
- States with `"End": true` are terminal — the execution ends at them.
- `Succeed` and `Fail` are always terminal (they have no `Next`).
- `Choice` is the only type that doesn't have `Next` at the top level — it defines `Choices[]` with `Next` in each rule.
- An execution *fails* if it reaches a `Fail` state or if an unhandled error occurs.
- An execution *succeeds* if it reaches `Succeed` or any state with `"End": true`.

---

### 3. Task State — invoking services and waiting for results

[FACT] The `Task` state is the heart of any workflow: it represents a unit of work delegated to an external service. The `Resource` field is an ARN that specifies what will be invoked.

Three distinct integration patterns:

```
PATTERN 1 — Request-Response (default)
───────────────────────────────────────────────────────────────────────────────
Step Functions invokes the service and advances IMMEDIATELY after the call is accepted.
Does not wait for the result. Suitable when the result doesn't matter or is asynchronous.

"Resource": "arn:aws:states:::lambda:invoke"


PATTERN 2 — Synchronous (suffix .sync:2)
───────────────────────────────────────────────────────────────────────────────
Step Functions waits for the job/operation to complete before advancing.
AWS SDK Integrations use internal polling via CloudWatch Events.

"Resource": "arn:aws:states:::lambda:invoke.waitForTaskToken"
             arn:aws:states:::dynamodb:putItem.sync:2
             arn:aws:states:::ecs:runTask.sync:2
             arn:aws:states:::glue:startJobRun.sync

PATTERN 3 — Wait for Task Token
───────────────────────────────────────────────────────────────────────────────
Step Functions stops and waits until an external worker calls
SendTaskSuccess or SendTaskFailure with the received token.
Suitable for human approvals, legacy processes, callbacks.

"Resource": "arn:aws:states:::sqs:sendMessage.waitForTaskToken"
```

[FACT] Example of Task invoking Lambda with `.waitForTaskToken`:

```json
"WaitForApproval": {
  "Type": "Task",
  "Resource": "arn:aws:states:::sqs:sendMessage.waitForTaskToken",
  "Parameters": {
    "QueueUrl": "https://sqs.us-east-1.amazonaws.com/123/approvals",
    "MessageBody": {
      "TaskToken.$": "$$.Task.Token",
      "Input.$": "$"
    }
  },
  "TimeoutSeconds": 86400,
  "HeartbeatSeconds": 3600,
  "Next": "ProcessResponse"
}
```

`$$.Task.Token` is a special reference to the **Context Object** — data about the current execution that Step Functions injects. The `$$` prefix accesses the context; `$` accesses the state input.

[FACT] Important optional fields in the Task State:

| Field | Description |
|---|---|
| `TimeoutSeconds` | Maximum wait time before throwing `States.Timeout` |
| `HeartbeatSeconds` | Maximum interval between heartbeats; if exceeded, throws `States.HeartbeatTimeout` |
| `Retry` | Array of retry rules with `ErrorEquals`, `IntervalSeconds`, `MaxAttempts`, `BackoffRate` |
| `Catch` | Array of fallbacks: if error matches `ErrorEquals`, transitions to `Next` state |
| `ResultPath` | Where in the input to write the result (e.g., `"$.result"` to not overwrite input) |
| `Parameters` | Reformats input before passing to the service (supports `.$` for references) |
| `ResultSelector` | Filters/reformats service output before writing |

---

### 4. Choice State — conditional logic

[FACT] The `Choice` state evaluates a list of rules in order and transitions to the first `Next` whose conditions are true. If no rule matches, it uses `Default` (mandatory in practice to avoid execution failure).

```json
"CheckStatus": {
  "Type": "Choice",
  "Choices": [
    {
      "Variable": "$.status",
      "StringEquals": "APPROVED",
      "Next": "ProcessApproved"
    },
    {
      "Variable": "$.attempts",
      "NumericGreaterThanEquals": 3,
      "Next": "EscalateToHuman"
    },
    {
      "And": [
        { "Variable": "$.amount", "NumericGreaterThan": 1000 },
        { "Variable": "$.category", "StringEquals": "VIP" }
      ],
      "Next": "ProcessVIP"
    }
  ],
  "Default": "ProcessDefault"
}
```

[FACT] Comparison operators available in ASL (selection):

```
StringEquals / StringEqualsPath
StringLessThan / StringGreaterThan / StringMatches (glob, e.g.: "foo*")
NumericEquals / NumericLessThan / NumericGreaterThan / ...
BooleanEquals / BooleanEqualsPath
TimestampEquals / TimestampLessThan / TimestampGreaterThan / ...
IsNull / IsPresent / IsString / IsNumeric / IsBoolean / IsTimestamp
And / Or / Not  (logical combinators)
```

[CONSENSUS] The `Path` suffix (e.g., `StringEqualsPath`) allows comparing the `Variable` value with the value of *another path* in the input — useful when the comparison value also comes from dynamic input.

```
IMPORTANT: Choice does not have "Next" at the top level.
Each rule within "Choices[]" has its own "Next".
Choice also cannot have "End: true".
```

---

### 5. Wait State — temporal pauses

[FACT] The `Wait` state pauses execution without consuming compute resources. Standard Workflow billing continues (state transitioned upon entering Wait), but no Lambda or external service is running.

Four ways to specify the duration:

```json
// 1. Fixed duration in seconds
{
  "Type": "Wait",
  "Seconds": 300,
  "Next": "NextState"
}

// 2. Fixed absolute timestamp
{
  "Type": "Wait",
  "Timestamp": "2026-12-31T23:59:59Z",
  "Next": "NextState"
}

// 3. Duration from input (JsonPath)
{
  "Type": "Wait",
  "SecondsPath": "$.wait_seconds",
  "Next": "NextState"
}

// 4. Timestamp from input (JsonPath)
{
  "Type": "Wait",
  "TimestampPath": "$.processing_date",
  "Next": "NextState"
}
```

[CONSENSUS] `TimestampPath` is the most powerful pattern: it allows scheduling execution continuation for a specific instant calculated in previous states. Example: send a reminder 24h after the user creates an order, using the timestamp generated by the creation Lambda.

---

### 6. Succeed and Fail States — terminal states

[FACT] `Succeed` ends the execution with status `SUCCEEDED`. It has no `Next`, no required fields beyond `Type`.

```json
"Completed": {
  "Type": "Succeed"
}
```

[FACT] `Fail` ends the execution with status `FAILED`. It accepts `Error` and `Cause` to describe the reason:

```json
"BusinessError": {
  "Type": "Fail",
  "Error": "InvalidOrder",
  "Cause": "Order amount exceeds available credit limit"
}
```

[FACT] It's also possible to use `ErrorPath` and `CausePath` (JsonPath) to populate these fields dynamically from the state input — useful when a Task's Catch forwards the error to a descriptive Fail state.

---

### 7. Execution History — reading and debugging executions

[FACT] For Standard Workflows, the `GetExecutionHistory` API returns all events of an execution in chronological order. Each event has `type`, `timestamp`, and type-specific payload.

Most relevant event types for diagnosis:

```
ExecutionStarted           — initial execution input
TaskStateEntered           — when a Task state was started
TaskScheduled              — when the external service was invoked
TaskStarted                — confirmation that the service started
TaskSucceeded              — successful service result
TaskFailed                 — error returned by the service
TaskTimedOut               — TimeoutSeconds or HeartbeatSeconds exceeded
ChoiceStateEntered         — Choice state being evaluated
WaitStateEntered/Exited    — start and end of wait
ExecutionFailed            — execution ended with failure (includes Error and Cause)
ExecutionSucceeded         — execution ended successfully
```

[FACT] For Express Workflows, `GetExecutionHistory` **is not available**. Events must be sent to CloudWatch Logs, configuring `logging_configuration` with `level` `ALL`, `ERROR`, `FATAL`, or `OFF`. Analysis is done via CloudWatch Log Insights.

---

## Practical example

**Scenario:** E-commerce order processing workflow. An order arrives, is validated by a Lambda, waits 2 hours for anti-fraud, is approved or rejected via Choice, and completes.

### Complete ASL definition

```json
{
  "Comment": "Order processing workflow",
  "StartAt": "ValidateOrder",
  "States": {

    "ValidateOrder": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "arn:aws:lambda:us-east-1:123456789:function:ValidateOrder",
        "Payload.$": "$"
      },
      "ResultSelector": {
        "valid.$": "$.Payload.valido",
        "fraud_score.$": "$.Payload.score_fraude",
        "order_id.$": "$.Payload.pedido_id"
      },
      "ResultPath": "$.validation",
      "TimeoutSeconds": 30,
      "Retry": [
        {
          "ErrorEquals": ["Lambda.ServiceException", "Lambda.TooManyRequestsException"],
          "IntervalSeconds": 2,
          "MaxAttempts": 3,
          "BackoffRate": 2
        }
      ],
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "ResultPath": "$.error",
          "Next": "ValidationFailure"
        }
      ],
      "Next": "WaitForAntifraud"
    },

    "WaitForAntifraud": {
      "Type": "Wait",
      "Seconds": 7200,
      "Next": "CheckFraud"
    },

    "CheckFraud": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.validation.valid",
          "BooleanEquals": false,
          "Next": "OrderRejected"
        },
        {
          "Variable": "$.validation.fraud_score",
          "NumericGreaterThan": 80,
          "Next": "SuspiciousOrder"
        }
      ],
      "Default": "ApproveOrder"
    },

    "ApproveOrder": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "arn:aws:lambda:us-east-1:123456789:function:ApproveOrder",
        "Payload.$": "$"
      },
      "ResultPath": null,
      "Next": "OrderApproved"
    },

    "OrderApproved": {
      "Type": "Succeed"
    },

    "SuspiciousOrder": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sqs:sendMessage.waitForTaskToken",
      "Parameters": {
        "QueueUrl": "https://sqs.us-east-1.amazonaws.com/123/manual-review",
        "MessageBody": {
          "TaskToken.$": "$$.Task.Token",
          "OrderId.$": "$.validation.order_id",
          "FraudScore.$": "$.validation.fraud_score"
        }
      },
      "TimeoutSeconds": 86400,
      "Next": "ApproveOrder"
    },

    "OrderRejected": {
      "Type": "Fail",
      "Error": "InvalidOrder",
      "Cause": "Order rejected during validation or fraud analysis"
    },

    "ValidationFailure": {
      "Type": "Fail",
      "Error": "ValidationError",
      "CausePath": "$.error.Cause"
    }

  }
}
```

### Flow diagram

```
                   ┌──────────────┐
                   │ ValidateOrder│ ◄─── Lambda invoke + Retry
                   └──────┬───────┘
                          │ ResultPath: $.validation
                          ▼
               ┌──────────────────────┐
               │ WaitForAntifraud     │ (Wait 2h)
               └──────────┬───────────┘
                          ▼
               ┌──────────────────────┐
               │    CheckFraud        │ (Choice)
               └──┬──────────┬────────┘
                  │          │           └──────────────────────┐
             valid=false   score>80       (default)             │
                  │          │                                   │
                  ▼          ▼                                   ▼
         ┌──────────┐  ┌───────────────┐              ┌──────────────────┐
         │ Order    │  │SuspiciousOrder│ (waitForToken)│  ApproveOrder    │
         │ Rejected │  │ → SQS → Manual│──────────────►│  Lambda invoke   │
         │ (Fail)   │  └───────────────┘              └────────┬─────────┘
         └──────────┘                                          ▼
                                                     ┌──────────────────┐
                                                     │  OrderApproved   │
                                                     │   (Succeed)      │
                                                     └──────────────────┘
```

### CDK — creating the state machine

```python
from aws_cdk import (
    Stack,
    aws_stepfunctions as sfn,
    aws_stepfunctions_tasks as tasks,
    aws_lambda as lambda_,
    Duration,
)
import json

class OrderWorkflowStack(Stack):

    def __init__(self, scope, construct_id, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # Lambda functions (already existing or defined here)
        fn_validate = lambda_.Function.from_function_name(
            self, "FnValidate", "ValidateOrder"
        )
        fn_approve = lambda_.Function.from_function_name(
            self, "FnApprove", "ApproveOrder"
        )

        # ── States ──────────────────────────────────────────────────
        validation_failure = sfn.Fail(
            self, "ValidationFailure",
            error="ValidationError",
            cause="Error validating order",
        )

        order_rejected = sfn.Fail(
            self, "OrderRejected",
            error="InvalidOrder",
            cause="Rejected in fraud analysis",
        )

        order_approved = sfn.Succeed(self, "OrderApproved")

        approve_order = tasks.LambdaInvoke(
            self, "ApproveOrder",
            lambda_function=fn_approve,
            result_path=sfn.JsonPath.DISCARD,  # ResultPath: null
        ).next(order_approved)

        check_fraud = sfn.Choice(self, "CheckFraud") \
            .when(
                sfn.Condition.boolean_equals("$.validation.valid", False),
                order_rejected
            ) \
            .when(
                sfn.Condition.number_greater_than("$.validation.fraud_score", 80),
                # SuspiciousOrder simplified for the CDK example
                approve_order
            ) \
            .otherwise(approve_order)

        wait_for_antifraud = sfn.Wait(
            self, "WaitForAntifraud",
            time=sfn.WaitTime.duration(Duration.hours(2)),
        ).next(check_fraud)

        validate_order = tasks.LambdaInvoke(
            self, "ValidateOrder",
            lambda_function=fn_validate,
            result_selector={
                "valid": sfn.JsonPath.string_at("$.Payload.valido"),
                "fraud_score": sfn.JsonPath.number_at("$.Payload.score_fraude"),
                "order_id": sfn.JsonPath.string_at("$.Payload.pedido_id"),
            },
            result_path="$.validation",
            timeout=Duration.seconds(30),
        ).add_retry(
            errors=["Lambda.ServiceException", "Lambda.TooManyRequestsException"],
            interval=Duration.seconds(2),
            max_attempts=3,
            backoff_rate=2,
        ).add_catch(
            validation_failure,
            errors=["States.ALL"],
            result_path="$.error",
        ).next(wait_for_antifraud)

        # ── State Machine ─────────────────────────────────────────────
        state_machine = sfn.StateMachine(
            self, "OrderProcessing",
            state_machine_name="OrderProcessing",
            definition_body=sfn.DefinitionBody.from_chainable(validate_order),
            # For Standard Workflow (default):
            state_machine_type=sfn.StateMachineType.STANDARD,
        )
```

### Execute and inspect via CLI

```bash
# Start execution
aws stepfunctions start-execution \
  --state-machine-arn arn:aws:states:us-east-1:123456789:stateMachine:OrderProcessing \
  --name "exec-order-001" \
  --input '{"order_id": "P001", "amount": 1500, "customer_id": "C42"}'

# List recent executions
aws stepfunctions list-executions \
  --state-machine-arn arn:aws:states:us-east-1:123456789:stateMachine:OrderProcessing \
  --status-filter RUNNING

# Describe an execution (status + input/output)
aws stepfunctions describe-execution \
  --execution-arn arn:aws:states:us-east-1:123456789:execution:OrderProcessing:exec-order-001

# View complete history (Standard Workflow only)
aws stepfunctions get-execution-history \
  --execution-arn arn:aws:states:us-east-1:123456789:execution:OrderProcessing:exec-order-001 \
  --query 'events[?type==`TaskFailed` || type==`ExecutionFailed`]'
```

---

## Common pitfalls

### Pitfall 1 — Confusing `ResultPath: null` with absence of `ResultPath`

**The mistake:** The developer wants the Task output to not alter the current state and simply omits `ResultPath`. The result is that the service output **completely replaces** the state input — all previous fields are lost.

**Why it happens:** The default behavior of `ResultPath` when absent is `$` — meaning the service result becomes the new input for the next state, overwriting everything.

**How to avoid:**
- `ResultPath: null` → discards the result, passes the original input forward.
- `ResultPath: "$.result"` → preserves the input and **adds** the result in the `.result` field.
- `ResultPath: "$"` (default) → replaces the entire input with the result.

```json
// Wrong: without ResultPath, Lambda output replaces $.order_id, $.amount etc.
"ApproveOrder": { "Type": "Task", "Resource": "...", "Next": "..." }

// Correct: discards result, preserves input
"ApproveOrder": { "Type": "Task", "Resource": "...", "ResultPath": null, "Next": "..." }
```

---

### Pitfall 2 — Using Express Workflow where the Task is not idempotent

**The mistake:** The developer chooses Express Workflow (cheaper, higher throughput) for a workflow that includes a Task creating a unique resource (e.g., inserting a record in a database with auto-increment ID). In case of internal service retry (at-least-once), the same record is created twice.

**Why it happens:** Asynchronous Express Workflows have at-least-once guarantee, not exactly-once. The developer often doesn't realize that "at-least-once" means the execution *may* run again.

**How to avoid:**
- For workflows with non-idempotent Tasks: use Standard Workflow.
- If you want to use Express for cost, ensure all Tasks are idempotent (upsert instead of insert, unique keys via `$.input.id`, etc.).
- Synchronous Express Workflow has at-most-once guarantee — but that means it may *not execute* on failures, not that it will execute exactly once.

---

### Pitfall 3 — Forgetting `TimeoutSeconds` on Tasks with `.waitForTaskToken`

**The mistake:** The `WaitForTaskToken` state is configured without `TimeoutSeconds`. The workflow stays blocked indefinitely if the worker that should send the token fails silently (crash, bug, unhandled error).

**Why it happens:** Step Functions doesn't impose a timeout by default — a Task without `TimeoutSeconds` waits up to the workflow's maximum limit (1 year in Standard). The developer tests the happy path and never notices the failure behavior.

**How to avoid:**
- **Always** define `TimeoutSeconds` on Tasks with `waitForTaskToken`. Use a realistic value for the process SLA (e.g., 86400 for 24h approvals).
- Also define `HeartbeatSeconds` if the worker should confirm progress periodically.
- Configure a `Catch` for `States.Timeout` and `States.HeartbeatTimeout` that transitions to a compensation or alert state.

```json
"WaitingForApproval": {
  "Type": "Task",
  "Resource": "arn:aws:states:::sqs:sendMessage.waitForTaskToken",
  "TimeoutSeconds": 86400,
  "HeartbeatSeconds": 3600,
  "Catch": [
    {
      "ErrorEquals": ["States.Timeout", "States.HeartbeatTimeout"],
      "Next": "ApprovalExpired"
    }
  ]
}
```

---

## Reflection exercise

You are designing a corporate customer onboarding system. The process involves: (1) validating registration data via internal API, (2) sending documents for compliance analysis (which can take up to 5 business days), (3) waiting for manual approval from an analyst, (4) provisioning system access when approved, or (5) notifying the customer and terminating if rejected.

**Question:** How would you choose between Standard Workflow and Express Workflow for this case? Which states would you use at each step (Task, Wait, Choice, Succeed, Fail)? In particular, how would you model step 3 (manual approval) so the analyst can approve or reject via a REST API without the execution blocking a server? Also describe how you would read the history of an execution that failed at step 4 to identify the root cause.

---

## Resources for further study

1. **AWS Step Functions Developer Guide — Welcome**
   URL: https://docs.aws.amazon.com/step-functions/latest/dg/welcome.html
   The entry point for the official documentation. The "Concepts" section covers terminology (state machine, execution, state, transition) with precision. Read especially "How Step Functions Works" to understand the execution model.

2. **Choosing workflow type — Standard vs Express**
   URL: https://docs.aws.amazon.com/step-functions/latest/dg/concepts-standard-vs-express.html
   The official comparative table with all parameters: guarantees, duration, pricing, logging, throughput, and use cases. It's the primary reference for workflow type decisions.

3. **Amazon States Language specification**
   URL: https://states-language.net/spec.html
   The complete and open ASL specification. Covers each state type, JsonPath in the ASL context (including Context Object with `$$`), required and optional fields, Choice operators, and error handling behavior. More precise than any tutorial for understanding edge cases.

---
