# Session 23 — Step Functions: Parallel, Map, data flow between states and error handling

**Estimated duration:** 60 minutes
**Prerequisites:** session-022-stepfunctions-standard-express-states

---

## Objective

By the end, you will be able to use Parallel for simultaneous execution of independent branches and Map for iteration over arrays (inline vs distributed mode), master the InputPath, OutputPath, ResultPath, and Parameters fields to control data flow between states without surprises, and implement Retry with exponential backoff and Catch by error type with redirection to a fallback state.

---

## Context

[FACT] The previous session covered the basic Step Functions states (Task, Choice, Wait, Succeed, Fail) and the differences between Standard and Express Workflows. This session advances to the three mechanisms that make Step Functions useful in real production scenarios: structured parallelism (Parallel), iteration over collections (Map), and declarative failure handling (Retry + Catch).

[CONSENSUS] Data flow between states — controlled by the InputPath, Parameters, ResultSelector, ResultPath, and OutputPath fields — is consistently pointed out by the community as the most confusing part of Step Functions. Most bugs in real workflows come from developers who don't understand the application order of these filters, resulting in data being silently lost or overwritten. Mastering this pipeline is as important as knowing the state types.

[FACT] The `Map` state gained a second mode of operation — **Distributed** — that allows up to 10,000 parallel iterations running as independent child executions. This mode was introduced in 2022 and is the basis for large-scale data processing pipelines directly in Step Functions, without the need for external tools like AWS Glue for simple orchestration.

---

## Core concepts

### 1. Parallel State — concurrent branches with aggregated output

[FACT] The `Parallel` state executes multiple branches (sub-workflows) simultaneously. Each branch is a complete mini state machine with `StartAt` and `States`. Step Functions waits for **all branches** to reach a terminal state before advancing to the Parallel's `Next`.

```
Parallel State
───────────────────────────────────────────────────────────────────────
                    ┌──── Branch 1 ─────┐
Input ──► Parallel ─┤                   ├──► Output (array[branch1, branch2, branch3])
                    ├──── Branch 2 ─────┤
                    └──── Branch 3 ─────┘
                    (all run at the same time)
                    (waits for the slowest)
```

[FACT] The Parallel output is always an **array** with one element per branch, in the order they were declared. Each branch's output is the output of its last state.

```json
"EnrichOrder": {
  "Type": "Parallel",
  "Branches": [
    {
      "StartAt": "FetchCustomer",
      "States": {
        "FetchCustomer": {
          "Type": "Task",
          "Resource": "arn:aws:states:::lambda:invoke",
          "Parameters": {
            "FunctionName": "FetchCustomer",
            "Payload.$": "$"
          },
          "ResultSelector": { "customer.$": "$.Payload" },
          "End": true
        }
      }
    },
    {
      "StartAt": "FetchInventory",
      "States": {
        "FetchInventory": {
          "Type": "Task",
          "Resource": "arn:aws:states:::lambda:invoke",
          "Parameters": {
            "FunctionName": "FetchInventory",
            "Payload.$": "$"
          },
          "ResultSelector": { "inventory.$": "$.Payload" },
          "End": true
        }
      }
    }
  ],
  "ResultPath": "$.enrichment",
  "Next": "ProcessEnrichedOrder"
}
```

After the Parallel above, `$.enrichment` will be:
```json
[
  { "customer": { "name": "...", "credit": 5000 } },
  { "inventory": { "sku": "P001", "available": 12 } }
]
```

[FACT] If **any branch** fails with an unhandled error within the branch, the entire Parallel state fails immediately (the other branches are cancelled). The Parallel can have its own `Retry` and `Catch` fields to handle failures from any branch.

[CONSENSUS] A common pitfall is expecting each branch to receive different input. In reality, all branches receive the **same input** — the effective input of the Parallel state (after InputPath/Parameters). To pass different data to each branch, transform the input within the first state of each branch.

---

### 2. Map State — iteration over arrays (Inline vs Distributed)

[FACT] The `Map` state iterates over an array (in the input or from an external source) and executes a sub-workflow for each item. The two modes have radically different characteristics:

```
┌──────────────────┬───────────────────────────┬────────────────────────────────┐
│ Dimension        │ Inline Map                │ Distributed Map                │
├──────────────────┼───────────────────────────┼────────────────────────────────┤
│ MaxConcurrency   │ Up to 40                  │ Up to 10,000                   │
├──────────────────┼───────────────────────────┼────────────────────────────────┤
│ Item source      │ Array in input JSON       │ JSON array, CSV or S3 list     │
├──────────────────┼───────────────────────────┼────────────────────────────────┤
│ Child execution  │ Inline (same execution)   │ Child workflow (own execution) │
├──────────────────┼───────────────────────────┼────────────────────────────────┤
│ Execution history│ Embedded in parent        │ Separate per iteration         │
├──────────────────┼───────────────────────────┼────────────────────────────────┤
│ Results          │ Array in Map output       │ Can write to S3                │
├──────────────────┼───────────────────────────┼────────────────────────────────┤
│ Failure tolerance│ ToleratedFailureCount/    │ ToleratedFailureCount/         │
│                  │ Percentage                │ Percentage                     │
└──────────────────┴───────────────────────────┴────────────────────────────────┘
```

#### Map Inline

```json
"ProcessItems": {
  "Type": "Map",
  "ItemsPath": "$.items",
  "ItemSelector": {
    "item_id.$": "$$.Map.Item.Value.id",
    "index.$": "$$.Map.Item.Index",
    "context.$": "$.global_context"
  },
  "MaxConcurrency": 10,
  "ToleratedFailurePercentage": 20,
  "Iterator": {
    "StartAt": "ProcessOneItem",
    "States": {
      "ProcessOneItem": {
        "Type": "Task",
        "Resource": "arn:aws:states:::lambda:invoke",
        "Parameters": {
          "FunctionName": "ProcessItem",
          "Payload.$": "$"
        },
        "ResultSelector": { "result.$": "$.Payload" },
        "End": true
      }
    }
  },
  "ResultPath": "$.results",
  "Next": "Finalize"
}
```

[FACT] Important Map fields:

| Field | Description |
|---|---|
| `ItemsPath` | JsonPath to the array in input. Default: `$` (entire input must be array) |
| `ItemSelector` | Builds the input for each iteration. `$$.Map.Item.Value` = current item; `$$.Map.Item.Index` = index |
| `MaxConcurrency` | Maximum parallel iterations. `0` = no limit (up to 40 in Inline) |
| `ToleratedFailureCount` | Absolute number of tolerated failures before the Map fails |
| `ToleratedFailurePercentage` | Percentage of tolerated failures (0-100) |

[FACT] `$$.Map.Item.Value` and `$$.Map.Item.Index` are **Context Object** references specific to Map — accessible only within `ItemSelector`. `$$` accesses the execution context; `$` accesses the item already transformed by ItemSelector.

#### Map Distributed

```json
"ProcessLargeCSV": {
  "Type": "Map",
  "ItemReader": {
    "Resource": "arn:aws:states:::s3:getObject",
    "ReaderConfig": {
      "InputType": "CSV",
      "CSVHeaderLocation": "FIRST_ROW"
    },
    "Parameters": {
      "Bucket": "my-bucket",
      "Key": "data/orders.csv"
    }
  },
  "MaxConcurrency": 1000,
  "ToleratedFailurePercentage": 5,
  "ItemSelector": {
    "row.$": "$$.Map.Item.Value"
  },
  "ItemBatcher": {
    "MaxItemsPerBatch": 10,
    "MaxInputBytesPerBatch": 262144
  },
  "Iterator": {
    "StartAt": "ProcessBatch",
    "States": {
      "ProcessBatch": {
        "Type": "Task",
        "Resource": "arn:aws:states:::lambda:invoke",
        "Parameters": {
          "FunctionName": "ProcessBatch",
          "Payload.$": "$"
        },
        "End": true
      }
    }
  },
  "ResultWriter": {
    "Resource": "arn:aws:states:::s3:putObject",
    "Parameters": {
      "Bucket": "my-bucket",
      "Prefix": "results/"
    }
  },
  "Next": "Finalize"
}
```

[FACT] `ItemBatcher` is exclusive to Distributed mode: it groups multiple items from the array into a single input for each iteration, reducing the number of invocations and increasing efficiency for loads with many small items.

[FACT] `ResultWriter` writes the results of all iterations to S3 instead of returning them inline in the Map output — essential when the volume of results would exceed the 256KB state payload limit.

---

### 3. Data pipeline between states — order matters

[FACT] Step Functions applies five filters in sequence before passing control to the next state. Understanding this order is critical to not losing data or corrupting the flow:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
DATA PIPELINE IN A STATE

  Raw Input (output of the previous state)
       │
       ▼
  ┌──────────┐
  │InputPath │  Filters the raw input. Default: "$" (everything).
  │          │  null → passes {} empty to Parameters.
  └────┬─────┘
       │ Effective Input
       ▼
  ┌────────────┐
  │ Parameters │  Builds the payload sent to the service.
  │            │  Keys with ".$" are JsonPath references in the Effective Input.
  └─────┬──────┘
        │ Task Input
        ▼
  ╔═════════╗
  ║ SERVICE  ║  Lambda / DynamoDB / SQS / etc.
  ╚═════╤═══╝
        │ Raw Result (service response)
        ▼
  ┌────────────────┐
  │ ResultSelector │  Filters/reshapes the raw result.
  │                │  Keys with ".$" are references in the Raw Result.
  └───────┬────────┘
          │ Selected Result
          ▼
  ┌────────────┐
  │ ResultPath │  Where to write the Selected Result in the Effective Input.
  │            │  "$"  → replaces the entire Effective Input.
  │            │  null → discards the result, passes Effective Input.
  │            │  "$.x"→ adds/overwrites the "x" field.
  └──────┬─────┘
         │ Effective Output
         ▼
  ┌────────────┐
  │ OutputPath │  Filters the Effective Output before sending.
  │            │  Default: "$" (everything). null → passes {}.
  └──────┬─────┘
         │
         ▼
  Input of the next state
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

[FACT] Concrete example to fix each step:

```json
// Raw input received by the state:
// { "order": { "id": "P001", "amount": 1500 }, "context": { "user": "U42" } }

{
  "Type": "Task",
  "Resource": "arn:aws:states:::lambda:invoke",

  // 1. InputPath: uses only the "order" field as effective input
  "InputPath": "$.order",
  // effective input: { "id": "P001", "amount": 1500 }

  // 2. Parameters: builds the payload for Lambda
  "Parameters": {
    "FunctionName": "ValidateOrder",
    "Payload": {
      "order_id.$": "$.id",
      "amount.$": "$.amount",
      "timestamp.$": "$$.Execution.StartTime"
    }
  },
  // task input to Lambda: { "order_id": "P001", "amount": 1500, "timestamp": "2026-06-22T..." }

  // 3. [Lambda executes and returns]:
  // { "StatusCode": 200, "Payload": { "approved": true, "score": 95 } }

  // 4. ResultSelector: takes only what matters from the Lambda result
  "ResultSelector": {
    "approved.$": "$.Payload.approved",
    "score.$":    "$.Payload.score"
  },
  // selected result: { "approved": true, "score": 95 }

  // 5. ResultPath: writes to the ORIGINAL effective input (InputPath=$.order)
  //    NOTE: ResultPath is applied on the effective input (after InputPath),
  //    not on the raw input.
  "ResultPath": "$.validation",
  // effective output: { "id": "P001", "amount": 1500, "validation": { "approved": true, "score": 95 } }

  // 6. OutputPath: sends only what's necessary
  "OutputPath": "$.validation",
  // output to next state: { "approved": true, "score": 95 }

  "Next": "CheckApproval"
}
```

[FACT] Critical point frequently confused: `ResultPath` is applied on the **effective input** (output of InputPath), not on the original raw input. If `InputPath` filtered to `$.order`, then the effective input no longer has `$.context`. When using `ResultPath: "$.validation"`, the result is added to the filtered `$.order`, not to the original complete input. To preserve the complete input, avoid using InputPath or use `InputPath: "$"`.

---

### 4. Error Handling — Retry and Catch

[FACT] Error handling in Step Functions is declarative: you specify, within the state, how to react to errors — without external code or retry lambdas. Retry is always evaluated before Catch.

#### Built-in error codes

[FACT] Step Functions defines a set of reserved errors with the `States.` prefix:

```
States.ALL                    — wildcard: matches any error
States.TaskFailed             — the Task returned failure
States.Timeout                — TimeoutSeconds exceeded
States.HeartbeatTimeout       — HeartbeatSeconds exceeded without heartbeat
States.NoChoiceMatched        — Choice without Default and no rule matched
States.ResultPathMatchFailure — ResultPath could not be applied
States.ParameterPathFailure   — JsonPath reference failed in Parameters
States.BranchFailed           — Parallel branch failed
States.ExceedToleratedFailureThreshold — Map exceeded failure tolerance
States.Runtime                — unclassified runtime error
```

External service errors follow the convention `<Service>.<ErrorType>`, for example:
- `Lambda.ServiceException` — Lambda internal error
- `Lambda.TooManyRequestsException` — throttling
- `Lambda.AWSLambdaException` — function execution error
- `DynamoDB.ProvisionedThroughputExceededException`

#### Retry — exponential backoff with jitter

[FACT] The `Retry` array is evaluated in order. The **first** `ErrorEquals` that matches is applied. Specific errors should come before `States.ALL`.

```json
"ProcessPayment": {
  "Type": "Task",
  "Resource": "arn:aws:states:::lambda:invoke",
  "Parameters": { "FunctionName": "ProcessPayment", "Payload.$": "$" },
  "Retry": [
    {
      "ErrorEquals": ["Lambda.TooManyRequestsException", "Lambda.ServiceException"],
      "IntervalSeconds": 1,
      "MaxAttempts": 5,
      "BackoffRate": 2.0,
      "MaxDelaySeconds": 30,
      "JitterStrategy": "FULL"
    },
    {
      "ErrorEquals": ["PaymentTemporarilyUnavailable"],
      "IntervalSeconds": 10,
      "MaxAttempts": 3,
      "BackoffRate": 1.5
    },
    {
      "ErrorEquals": ["States.ALL"],
      "IntervalSeconds": 5,
      "MaxAttempts": 2,
      "BackoffRate": 1.0
    }
  ],
  "Catch": [ ... ],
  "Next": "PaymentApproved"
}
```

[FACT] Retry fields:

| Field | Required | Default | Description |
|---|---|---|---|
| `ErrorEquals` | Yes | — | Array of error codes that activate this rule |
| `IntervalSeconds` | No | 1 | Initial wait before the first retry |
| `MaxAttempts` | No | 3 | Total attempts (0 = no retry) |
| `BackoffRate` | No | 2.0 | Multiplier applied on each retry |
| `MaxDelaySeconds` | No | no limit | Cap on maximum interval between retries |
| `JitterStrategy` | No | `NONE` | `FULL` adds random jitter (0 to calculated interval) |

[FACT] Interval calculation with backoff and `FULL` jitter:
```
Attempt 1: random(0, 1)   seconds    (IntervalSeconds=1, BackoffRate=2)
Attempt 2: random(0, 2)   seconds
Attempt 3: random(0, 4)   seconds
Attempt 4: random(0, 8)   seconds  → if MaxDelaySeconds=10, capped at 10
Attempt 5: random(0, 10)  seconds
```

[CONSENSUS] `JitterStrategy: FULL` is recommended when multiple parallel executions may fail simultaneously (e.g., multiple Lambdas hitting the same throttled service). Without jitter, all retries happen at the same intervals, creating load waves — the phenomenon called *thundering herd*.

#### Catch — fallback after retries exhausted

[FACT] After all retries of a rule fail, `Catch` is evaluated. The `Catch` array is also evaluated in order — first match wins.

```json
"Catch": [
  {
    "ErrorEquals": ["DuplicateOrder", "CreditLimit"],
    "ResultPath": "$.business_error",
    "Next": "HandleBusinessError"
  },
  {
    "ErrorEquals": ["States.Timeout"],
    "ResultPath": "$.timeout_error",
    "Next": "NotifyTimeout"
  },
  {
    "ErrorEquals": ["States.ALL"],
    "ResultPath": "$.error",
    "Next": "GenericFallback"
  }
]
```

[FACT] When Catch is triggered, Step Functions injects an error object with `Error` and `Cause` fields into the next state. The `ResultPath` controls where this object is inserted:

```json
// Without ResultPath in Catch (or ResultPath: "$"):
// Fallback state input = { "Error": "Lambda.ServiceException", "Cause": "..." }
// (original input is completely replaced)

// With ResultPath: "$.error":
// Fallback state input = { ...original_input..., "error": { "Error": "...", "Cause": "..." } }
// (original input preserved + error added)
```

[CONSENSUS] **Always** use `ResultPath` in Catch to preserve the original context. Without it, the fallback state receives only the error message, without the order/customer/etc. data that would be needed to log, compensate, or notify.

#### Complete flow: Retry → Catch

```
Task State is invoked
       │
       ▼ error occurs
┌──────────────┐
│ Retry[0]     │ → Checks ErrorEquals → Match?
│ ErrorEquals  │      Yes → waits IntervalSeconds → reinvokes → error → Retry[1]?
│ MaxAttempts  │                                              → success → Next
└──────┬───────┘
       │ MaxAttempts exhausted
       ▼
┌──────────────┐
│ Retry[1]     │ → next retry rule
└──────┬───────┘
       │ all retries exhausted
       ▼
┌──────────────┐
│ Catch[0]     │ → Checks ErrorEquals → Match → Next (fallback state)
│ Catch[1]     │
└──────┬───────┘
       │ no Catch matched
       ▼
  Execution FAILED
```

---

## Practical example

**Scenario:** Invoice processing pipeline. Receives an array of invoices, validates each in parallel with fiscal and customer enrichment, and processes the result.

```json
{
  "Comment": "Invoice processing pipeline",
  "StartAt": "ProcessInvoices",
  "States": {

    "ProcessInvoices": {
      "Type": "Map",
      "ItemsPath": "$.invoices",
      "ItemSelector": {
        "invoice.$": "$$.Map.Item.Value",
        "index.$": "$$.Map.Item.Index"
      },
      "MaxConcurrency": 20,
      "ToleratedFailurePercentage": 10,
      "Iterator": {
        "StartAt": "EnrichInvoice",
        "States": {

          "EnrichInvoice": {
            "Type": "Parallel",
            "Branches": [
              {
                "StartAt": "ValidateFiscal",
                "States": {
                  "ValidateFiscal": {
                    "Type": "Task",
                    "Resource": "arn:aws:states:::lambda:invoke",
                    "Parameters": {
                      "FunctionName": "ValidateFiscal",
                      "Payload.$": "$.invoice"
                    },
                    "ResultSelector": { "fiscal.$": "$.Payload" },
                    "Retry": [
                      {
                        "ErrorEquals": ["Lambda.TooManyRequestsException"],
                        "IntervalSeconds": 2,
                        "MaxAttempts": 3,
                        "BackoffRate": 2,
                        "JitterStrategy": "FULL"
                      }
                    ],
                    "End": true
                  }
                }
              },
              {
                "StartAt": "FetchCustomerData",
                "States": {
                  "FetchCustomerData": {
                    "Type": "Task",
                    "Resource": "arn:aws:states:::dynamodb:getItem.sync:2",
                    "Parameters": {
                      "TableName": "customers",
                      "Key": {
                        "tax_id": { "S.$": "$.invoice.issuer_tax_id" }
                      }
                    },
                    "ResultSelector": { "customer.$": "$.Item" },
                    "Catch": [
                      {
                        "ErrorEquals": ["DynamoDB.ResourceNotFoundException"],
                        "ResultPath": "$.customer_error",
                        "Next": "CustomerNotFound"
                      }
                    ],
                    "End": true
                  },
                  "CustomerNotFound": {
                    "Type": "Pass",
                    "Result": { "customer": null },
                    "End": true
                  }
                }
              }
            ],
            "ResultSelector": {
              "fiscal.$":   "$[0].fiscal",
              "customer.$": "$[1].customer"
            },
            "ResultPath": "$.enrichment",
            "Next": "CheckFraud"
          },

          "CheckFraud": {
            "Type": "Choice",
            "Choices": [
              {
                "Variable": "$.enrichment.fiscal.status",
                "StringEquals": "IRREGULAR",
                "Next": "BlockInvoice"
              }
            ],
            "Default": "RegisterInvoice"
          },

          "RegisterInvoice": {
            "Type": "Task",
            "Resource": "arn:aws:states:::dynamodb:putItem.sync:2",
            "Parameters": {
              "TableName": "processed_invoices",
              "Item": {
                "id":       { "S.$": "$.invoice.access_key" },
                "fiscal":   { "S.$": "States.JsonToString($.enrichment.fiscal)" },
                "status":   { "S": "PROCESSED" }
              }
            },
            "ResultPath": null,
            "Retry": [
              {
                "ErrorEquals": ["DynamoDB.ProvisionedThroughputExceededException"],
                "IntervalSeconds": 1,
                "MaxAttempts": 5,
                "BackoffRate": 2,
                "MaxDelaySeconds": 20,
                "JitterStrategy": "FULL"
              }
            ],
            "Catch": [
              {
                "ErrorEquals": ["States.ALL"],
                "ResultPath": "$.registration_error",
                "Next": "RegistrationFailure"
              }
            ],
            "End": true
          },

          "BlockInvoice": {
            "Type": "Fail",
            "Error": "IrregularInvoice",
            "Cause": "Invoice with irregular fiscal status"
          },

          "RegistrationFailure": {
            "Type": "Fail",
            "Error": "RegistrationFailure",
            "CausePath": "$.registration_error.Cause"
          }
        }
      },
      "ResultPath": "$.results",
      "Next": "GenerateReport"
    },

    "GenerateReport": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "GenerateReport",
        "Payload": {
          "total.$": "States.ArrayLength($.results)",
          "results.$": "$.results"
        }
      },
      "ResultPath": null,
      "End": true
    }

  }
}
```

### CDK — Parallel, Map and error handling in Python

```python
from aws_cdk import (
    Stack, Duration,
    aws_stepfunctions as sfn,
    aws_stepfunctions_tasks as tasks,
    aws_lambda as lambda_,
    aws_dynamodb as dynamodb,
)

class InvoiceWorkflowStack(Stack):

    def __init__(self, scope, construct_id, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        table = dynamodb.Table.from_table_name(self, "Table", "processed_invoices")
        fn_validate = lambda_.Function.from_function_name(self, "FnValidate", "ValidateFiscal")
        fn_report = lambda_.Function.from_function_name(self, "FnReport", "GenerateReport")

        # ── Parallel branches ──────────────────────────────────────
        validate_fiscal = tasks.LambdaInvoke(
            self, "ValidateFiscal",
            lambda_function=fn_validate,
            payload=sfn.TaskInput.from_json_path_at("$.invoice"),
            result_selector={"fiscal": sfn.JsonPath.object_at("$.Payload")},
        ).add_retry(
            errors=["Lambda.TooManyRequestsException"],
            interval=Duration.seconds(2),
            max_attempts=3,
            backoff_rate=2,
            jitter_strategy=sfn.JitterType.FULL,
        )

        customer_not_found = sfn.Pass(
            self, "CustomerNotFound",
            result=sfn.Result.from_object({"customer": None}),
        )

        fetch_customer = tasks.DynamoGetItem(
            self, "FetchCustomer",
            table=table,
            key={"tax_id": tasks.DynamoAttributeValue.from_string(
                sfn.JsonPath.string_at("$.invoice.issuer_tax_id")
            )},
            result_selector={"customer": sfn.JsonPath.object_at("$.Item")},
        ).add_catch(
            customer_not_found,
            errors=["DynamoDB.ResourceNotFoundException"],
            result_path="$.customer_error",
        )

        enrich = sfn.Parallel(
            self, "EnrichInvoice",
            result_selector={
                "fiscal":   sfn.JsonPath.object_at("$[0].fiscal"),
                "customer": sfn.JsonPath.object_at("$[1].customer"),
            },
            result_path="$.enrichment",
        )
        enrich.branch(validate_fiscal)
        enrich.branch(fetch_customer)

        # ── Fallbacks and terminal states ────────────────────────────
        block_invoice = sfn.Fail(
            self, "BlockInvoice",
            error="IrregularInvoice",
            cause="Invoice with irregular fiscal status",
        )

        registration_failure = sfn.Fail(
            self, "RegistrationFailure",
            error="RegistrationFailure",
        )

        # ── Register Invoice ──────────────────────────────────────────
        register_invoice = tasks.DynamoPutItem(
            self, "RegisterInvoice",
            table=table,
            item={
                "id":     tasks.DynamoAttributeValue.from_string(
                              sfn.JsonPath.string_at("$.invoice.access_key")),
                "status": tasks.DynamoAttributeValue.from_string("PROCESSED"),
            },
            result_path=sfn.JsonPath.DISCARD,
        ).add_retry(
            errors=["DynamoDB.ProvisionedThroughputExceededException"],
            interval=Duration.seconds(1),
            max_attempts=5,
            backoff_rate=2,
            max_delay=Duration.seconds(20),
            jitter_strategy=sfn.JitterType.FULL,
        ).add_catch(
            registration_failure,
            errors=["States.ALL"],
            result_path="$.registration_error",
        )

        # ── Choice ────────────────────────────────────────────────────
        check_fraud = sfn.Choice(self, "CheckFraud") \
            .when(
                sfn.Condition.string_equals("$.enrichment.fiscal.status", "IRREGULAR"),
                block_invoice,
            ) \
            .otherwise(register_invoice)

        # ── Map iterator ───────────────────────────────────────────
        iterator = enrich.next(check_fraud)

        # ── Main Map ─────────────────────────────────────────────
        process_invoices = sfn.Map(
            self, "ProcessInvoices",
            items_path=sfn.JsonPath.string_at("$.invoices"),
            item_selector={
                "invoice": sfn.JsonPath.object_at("$$.Map.Item.Value"),
                "index":   sfn.JsonPath.number_at("$$.Map.Item.Index"),
            },
            max_concurrency=20,
            tolerated_failure_percentage=10,
            result_path="$.results",
        ).iterator(iterator)

        generate_report = tasks.LambdaInvoke(
            self, "GenerateReport",
            lambda_function=fn_report,
            result_path=sfn.JsonPath.DISCARD,
        )

        definition = process_invoices.next(generate_report)

        sfn.StateMachine(
            self, "InvoiceWorkflow",
            state_machine_name="InvoiceProcessing",
            definition_body=sfn.DefinitionBody.from_chainable(definition),
            state_machine_type=sfn.StateMachineType.STANDARD,
        )
```

---

## Common pitfalls

### Pitfall 1 — `ResultPath` in Catch without preserving the original input causes context loss

**The mistake:** The developer configures a Catch without `ResultPath` (or with `ResultPath: "$"`). In the fallback state, the entire input is the object `{ "Error": "...", "Cause": "..." }` — the order, the customer, the execution context: everything is gone.

**Why it happens:** The default behavior of absent `ResultPath` in Catch is `"$"`, which replaces the entire effective input with the error object. It's the same behavior as Task, except here the "result" is the error.

**How to avoid:** Always use `"ResultPath": "$.error"` (or any reserved field) in Catch. The fallback state will receive the original input intact plus the `$.error` field with `{ "Error": "...", "Cause": "..." }`.

```json
// Wrong — loses context
"Catch": [{ "ErrorEquals": ["States.ALL"], "Next": "Fallback" }]

// Correct — preserves context
"Catch": [{ "ErrorEquals": ["States.ALL"], "ResultPath": "$.error", "Next": "Fallback" }]
```

---

### Pitfall 2 — Using `InputPath` when meaning `Parameters`, losing input fields

**The mistake:** The developer wants to pass only `$.order_id` to Lambda. They use `"InputPath": "$.order_id"`. The result: Lambda receives only the ID string (e.g., `"P001"`), not an object. Additionally, the effective input for the rest of the pipeline (ResultPath, etc.) is now also just that string.

**Why it happens:** `InputPath` filters the **entire effective input** — if the selected value is a string, the effective input becomes a string. `Parameters` should be used to build a structured payload without losing the input.

**How to avoid:**
- Use `InputPath` only when you want to **select a sub-tree** of the input that will become the new effective input (keeping the object).
- Use `Parameters` when you want to **build a new object** with fields selected from the input.

```json
// Wrong: effective input becomes the string "P001"
"InputPath": "$.order_id"

// Correct: effective input becomes { "id": "P001", "flag": true }
"Parameters": {
  "id.$": "$.order_id",
  "flag": true
}
```

---

### Pitfall 3 — Placing `States.ALL` before specific errors in Retry or Catch

**The mistake:** The Retry or Catch array starts with `{ "ErrorEquals": ["States.ALL"], ... }`. This causes **all** errors — including those that had specific treatment planned — to fall into the generic rule. Subsequent rules are never evaluated.

**Why it happens:** Both Retry and Catch evaluate rules in **order** and stop at the first match. `States.ALL` matches any error, so it blocks everything that follows.

**How to avoid:** Always place specific errors first, `States.ALL` last:

```json
// Wrong
"Retry": [
  { "ErrorEquals": ["States.ALL"], "MaxAttempts": 3 },
  { "ErrorEquals": ["Lambda.TooManyRequestsException"], "MaxAttempts": 10 }
]

// Correct
"Retry": [
  { "ErrorEquals": ["Lambda.TooManyRequestsException"], "MaxAttempts": 10, "BackoffRate": 2 },
  { "ErrorEquals": ["Lambda.ServiceException"], "MaxAttempts": 5, "BackoffRate": 1.5 },
  { "ErrorEquals": ["States.ALL"], "MaxAttempts": 2, "BackoffRate": 1 }
]
```

---

## Reflection exercise

You are modeling a data import workflow from a CSV file stored in S3. The file has ~50,000 rows. For each row, your team needs to: (1) enrich with data from an external API (with 200ms SLA and rate limit of 1,000 req/s), (2) validate business rules, and (3) persist to DynamoDB.

**Question:** How would you choose between Map Inline and Map Distributed for this case? What would be the appropriate `MaxConcurrency` to respect the external API rate limit? Where would you place the `ResultWriter` and why? Also describe how you would configure Retry for step 1 (external API call with rate limit) and Catch for step 3 (DynamoDB failure), so that rows with persistence errors are marked for reprocessing without canceling processing of the remaining ones.

---

## Resources for further study

1. **AWS Step Functions — Map State**
   URL: https://docs.aws.amazon.com/step-functions/latest/dg/state-map.html
   Entry point for Map state documentation, with links to both modes (Inline and Distributed). The "Inline Mode" section details ItemsPath, ItemSelector, and MaxConcurrency; the "Distributed Mode" section covers ItemReader, ItemBatcher, and ResultWriter with CSV and S3 examples.

2. **Processing input and output in Step Functions**
   URL: https://docs.aws.amazon.com/step-functions/latest/dg/concepts-input-output-filtering.html
   Complete reference for the InputPath → Parameters → ResultSelector → ResultPath → OutputPath pipeline. Includes visual examples of data state at each step, with tables showing before/after of each filter.

3. **Handling errors in Step Functions workflows**
   URL: https://docs.aws.amazon.com/step-functions/latest/dg/concepts-error-handling.html
   Complete list of `States.*` error codes, Retry semantics (fields, BackoffRate, JitterStrategy, MaxDelaySeconds) and Catch (evaluation order, interaction with ResultPath). Includes examples of states with both configured.

---
