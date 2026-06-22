# Session 034 — DynamoDB: transactions and conditional operations

**Estimated duration:** 60 minutes  
**Dependencies:** session-033-dax-arquitetura-casos-uso

---

## Objective

By the end of this session, you will be able to implement a `TransactWriteItems` that updates multiple entities atomically, use `ConditionExpression` for optimistic locking with a version counter, understand transaction limits (100 items, 4 MB, 2× cost), and correctly distinguish the isolation levels of each operation.

---

## Context

[FACT] DynamoDB offers native ACID transaction support since November 2018. `TransactWriteItems` and `TransactGetItems` provide atomicity and serializable isolation for operations that need consistency across multiple items — without the need for client-side coordination.

[CONSENSUS] The 2× cost of transactions (each item requires two internal operations: prepare + commit) is frequently misunderstood. For transactions with multiple items, this cost can be economically superior to trying to implement consistency on the application side with retries and manual rollbacks.

---

## Main concepts

### 1. TransactWriteItems: available actions and limits

[FACT] `TransactWriteItems` groups up to **100 write actions** in a single all-or-nothing operation. The allowed actions are:

```
Action          Equivalent         Typical use
─────────────────────────────────────────────────────────────
Put             PutItem             Create or replace an item
Update          UpdateItem          Modify attributes
Delete          DeleteItem          Remove an item
ConditionCheck  (no equivalent)     Verify condition without writing
                                    (e.g., verify sufficient balance
                                    before debiting from another item)
```

[FACT] Hard limits:
- Maximum **100 distinct items** per transaction
- Aggregate item size: maximum **4 MB**
- Only within the **same region** and **same AWS account**
- **Not possible** to target the same item twice in the same transaction (e.g., ConditionCheck + Update on the same item → validation error)

[FACT] **Transactions do not work via indexes** — actions must always reference the base table via PK (+ SK if it exists).

---

### 2. 2× cost and how to calculate

[FACT] Each item in a transaction consumes **2× the normal cost** of write or read, because DynamoDB executes two internal phases (prepare and commit):

```
Example: TransactWriteItems with 3 items of 500 bytes each

Normal calculation (without transaction):
  Item 1: ceil(500B / 1KB) = 1 WCU
  Item 2: 1 WCU
  Item 3: 1 WCU
  Total: 3 WCU

With transaction (2× per item):
  Item 1: 1 WCU × 2 = 2 WCU
  Item 2: 1 WCU × 2 = 2 WCU
  Item 3: 1 WCU × 2 = 2 WCU
  Total: 6 WCU

With transaction via DAX (extra RCU to populate cache):
  6 WCU (write) + 6 RCU (TransactGetItems post-write)
  Total: 6 WCU + 6 RCU
```

[FACT] The 2× cost appears in CloudWatch as `ConsumedWriteCapacityUnits` — the two internal operations (prepare and commit) are visible separately in metrics.

---

### 3. Isolation: levels table by operation

[FACT] The isolation levels between transactions and other operation types are:

```
Concurrent operation                  Isolation level
──────────────────────────────────────────────────────────────
PutItem / UpdateItem / DeleteItem     SERIALIZABLE
GetItem (individual)                  SERIALIZABLE
TransactWriteItems / TransactGetItems SERIALIZABLE (between themselves)
BatchGetItem (unit)                   READ-COMMITTED *
BatchWriteItem (unit)                 NOT serializable *
Query / Scan                          READ-COMMITTED

* individual items within batch/query = serializable;
  the batch/query as a unit = read-committed
```

[FACT] **Serializable**: the result is equivalent to executing operations one after another, without overlap. **Read-committed**: the read returns only values from already-completed transactions, but may see different versions of different items if a transaction completed between reads.

[FACT] After a successful `TransactWriteItems`, subsequent **eventually consistent** reads may still return the previous state for a short period. To guarantee the most recent value immediately after the transaction, use `ConsistentRead=True` via DynamoDB client (not DAX).

---

### 4. Idempotency with ClientRequestToken

[FACT] `TransactWriteItems` accepts an optional `ClientRequestToken` to guarantee idempotency in case of retries due to timeout or connectivity issues:

```
Idempotency window: 10 minutes
  ├── Same token, same parameters → success, without repeating the write
  └── Same token, different parameters → IdempotentParameterMismatch

After 10 minutes: same token is treated as a new request
```

[FACT] If `ReturnConsumedCapacity` is configured: the original call returns consumed WCUs; repeated calls within the window return RCUs (read cost to verify idempotency).

---

### 5. ConditionExpression: functions and operators

[FACT] `ConditionExpression` can be used in `PutItem`, `UpdateItem`, `DeleteItem`, and in the `ConditionCheck` action within transactions. If the condition fails, the operation returns `ConditionalCheckFailedException`.

```
Available functions:
─────────────────────────────────────────────────────────────
attribute_exists(path)         True if the attribute exists
attribute_not_exists(path)     True if the attribute does NOT exist
attribute_type(path, type)     Checks type: S, N, B, SS, NS, BS,
                               M, L, NULL, BOOL
begins_with(path, substr)      String starts with the given value
contains(path, operand)        Set contains element / string contains substr
size(path)                     Attribute size (number of bytes/chars)

Comparison operators: =  <>  <  <=  >  >=
Logical operators: AND  OR  NOT
Range operator: BETWEEN :lo AND :hi
List operator: IN (:v1, :v2, ...)
```

[FACT] `attribute_not_exists(PK)` in a `PutItem` is the canonical mechanism for **put-if-not-exists** — prevents overwriting an existing item. If the table has PK + SK, the condition checks if the **combination** PK+SK does not exist.

---

### 6. Optimistic locking with version counter

[FACT] The optimistic locking pattern in DynamoDB uses a `version` attribute (integer) that is checked before each write and incremented along with the update:

```
Optimistic locking flow:

1. Client A reads item: {PK: "X", version: 5, data: "abc"}
2. Client B reads item: {PK: "X", version: 5, data: "abc"}
3. Client A does UpdateItem:
   UpdateExpression:    "SET data = :new, version = version + :inc"
   ConditionExpression: "version = :expected"
   ExpressionAttributeValues: {":new": "xyz", ":inc": 1, ":expected": 5}
   → Success: item now has version: 6

4. Client B tries UpdateItem with version: 5
   ConditionExpression: "version = :expected" (expected 5, found 6)
   → ConditionalCheckFailedException: Client B must re-read and retry
```

[CONSENSUS] Optimistic locking is preferable to pessimistic locking (explicit locks) in DynamoDB because: (a) there is no native lock mechanism; (b) actual conflicts are rare in most workloads; (c) the retry cost is less than lock coordination overhead.

---

## Practical example

### Scenario: bank transfer — atomic debit and credit with balance ConditionCheck

**Table:** `accounts-table` (single-table design)  
**Operation:** transfer value between two accounts with balance validation, versioning and atomic auditing

#### CDK Python — Table and Lambda function

```python
from aws_cdk import (
    Stack, RemovalPolicy,
    aws_dynamodb as dynamodb,
    aws_lambda as lambda_,
    aws_iam as iam,
)
from constructs import Construct


class BankingStack(Stack):
    def __init__(self, scope: Construct, construct_id: str, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        accounts_table = dynamodb.Table(
            self, "AccountsTable",
            table_name="accounts-table",
            partition_key=dynamodb.Attribute(
                name="PK", type=dynamodb.AttributeType.STRING
            ),
            sort_key=dynamodb.Attribute(
                name="SK", type=dynamodb.AttributeType.STRING
            ),
            billing_mode=dynamodb.BillingMode.PAY_PER_REQUEST,
            removal_policy=RemovalPolicy.DESTROY,
        )

        transfer_fn = lambda_.Function(
            self, "TransferFn",
            function_name="bank-transfer",
            runtime=lambda_.Runtime.PYTHON_3_12,
            handler="handler.lambda_handler",
            code=lambda_.Code.from_asset("lambda/transfer"),
        )
        accounts_table.grant_read_write_data(transfer_fn)
```

#### Python — Complete implementation

```python
# lambda/transfer/handler.py
import uuid
from decimal import Decimal
from datetime import datetime, timezone
import boto3
from boto3.dynamodb.conditions import Attr
from botocore.exceptions import ClientError

dynamodb = boto3.resource("dynamodb", region_name="us-east-1")
table = dynamodb.Table("accounts-table")


# ── Setup operations (account creation) ───────────────────────────────

def create_account(account_id: str, owner: str, initial_balance: Decimal) -> dict:
    """
    PutItem with attribute_not_exists — put-if-not-exists.
    Fails with ConditionalCheckFailedException if account already exists.
    """
    try:
        table.put_item(
            Item={
                "PK": f"ACCOUNT#{account_id}",
                "SK": "METADATA",
                "accountId": account_id,
                "owner": owner,
                "balance": initial_balance,
                "version": 0,
                "status": "ACTIVE",
                "createdAt": datetime.now(timezone.utc).isoformat(),
            },
            ConditionExpression="attribute_not_exists(PK)",
        )
        return {"success": True, "accountId": account_id}
    except ClientError as e:
        if e.response["Error"]["Code"] == "ConditionalCheckFailedException":
            raise ValueError(f"Account {account_id} already exists")
        raise


def deactivate_account_if_empty(account_id: str) -> None:
    """
    Conditional UpdateItem: only deactivates if balance = 0.
    """
    try:
        table.update_item(
            Key={"PK": f"ACCOUNT#{account_id}", "SK": "METADATA"},
            UpdateExpression="SET #s = :inactive, updatedAt = :ts",
            ConditionExpression="balance = :zero AND #s = :active",
            ExpressionAttributeNames={"#s": "status"},  # "status" is a reserved word
            ExpressionAttributeValues={
                ":inactive": "INACTIVE",
                ":active": "ACTIVE",
                ":zero": Decimal("0"),
                ":ts": datetime.now(timezone.utc).isoformat(),
            },
        )
    except ClientError as e:
        if e.response["Error"]["Code"] == "ConditionalCheckFailedException":
            raise ValueError("Cannot deactivate: account not active or balance not zero")
        raise


# ── Atomic transfer with TransactWriteItems ───────────────────────────

def transfer(
    from_account_id: str,
    to_account_id: str,
    amount: Decimal,
    from_version: int,
    to_version: int,
    idempotency_key: str | None = None,
) -> dict:
    """
    Atomic transfer with:
    - ConditionCheck: validates from_account has sufficient balance and is ACTIVE
    - Update on from_account: decrements balance + increments version
    - Update on to_account: increments balance + increments version
    - Put of audit record (immutable transaction)

    from_version / to_version: versions previously read (optimistic locking)
    idempotency_key: optional, guarantees idempotency for 10 minutes
    """
    if amount <= 0:
        raise ValueError("Amount must be positive")

    transfer_id = str(uuid.uuid4())
    ts = datetime.now(timezone.utc).isoformat()

    try:
        kwargs = {
            "TransactItems": [
                # 1. ConditionCheck: sufficient balance + active account + correct version
                {
                    "ConditionCheck": {
                        "TableName": "accounts-table",
                        "Key": {
                            "PK": {"S": f"ACCOUNT#{from_account_id}"},
                            "SK": {"S": "METADATA"},
                        },
                        "ConditionExpression": (
                            "balance >= :amount "
                            "AND #s = :active "
                            "AND version = :from_v"
                        ),
                        "ExpressionAttributeNames": {"#s": "status"},
                        "ExpressionAttributeValues": {
                            ":amount": {"N": str(amount)},
                            ":active": {"S": "ACTIVE"},
                            ":from_v": {"N": str(from_version)},
                        },
                    }
                },
                # 2. Update: debit from source account
                {
                    "Update": {
                        "TableName": "accounts-table",
                        "Key": {
                            "PK": {"S": f"ACCOUNT#{from_account_id}"},
                            "SK": {"S": "METADATA"},
                        },
                        "UpdateExpression": (
                            "SET balance = balance - :amount, "
                            "version = version + :inc, "
                            "updatedAt = :ts"
                        ),
                        "ExpressionAttributeValues": {
                            ":amount": {"N": str(amount)},
                            ":inc": {"N": "1"},
                            ":ts": {"S": ts},
                        },
                    }
                },
                # 3. Update: credit to destination account (with versioning)
                {
                    "Update": {
                        "TableName": "accounts-table",
                        "Key": {
                            "PK": {"S": f"ACCOUNT#{to_account_id}"},
                            "SK": {"S": "METADATA"},
                        },
                        "UpdateExpression": (
                            "SET balance = balance + :amount, "
                            "version = version + :inc, "
                            "updatedAt = :ts"
                        ),
                        "ConditionExpression": "version = :to_v",
                        "ExpressionAttributeValues": {
                            ":amount": {"N": str(amount)},
                            ":inc": {"N": "1"},
                            ":to_v": {"N": str(to_version)},
                            ":ts": {"S": ts},
                        },
                    }
                },
                # 4. Put: immutable audit record
                {
                    "Put": {
                        "TableName": "accounts-table",
                        "Key": {
                            "PK": {"S": f"TRANSFER#{transfer_id}"},
                            "SK": {"S": "RECORD"},
                        },
                        "Item": {
                            "PK": {"S": f"TRANSFER#{transfer_id}"},
                            "SK": {"S": "RECORD"},
                            "fromAccount": {"S": from_account_id},
                            "toAccount": {"S": to_account_id},
                            "amount": {"N": str(amount)},
                            "createdAt": {"S": ts},
                            "status": {"S": "COMPLETED"},
                        },
                        # Guarantees transfer_id is unique (UUID collision guard)
                        "ConditionExpression": "attribute_not_exists(PK)",
                    }
                },
            ]
        }

        if idempotency_key:
            kwargs["ClientRequestToken"] = idempotency_key

        # Note: TransactWriteItems via low-level (client) not via resource
        client = boto3.client("dynamodb", region_name="us-east-1")
        client.transact_write_items(**kwargs)

        return {
            "success": True,
            "transferId": transfer_id,
            "amount": str(amount),
        }

    except ClientError as e:
        code = e.response["Error"]["Code"]

        if code == "TransactionCanceledException":
            # Extract cancellation reasons per item
            reasons = e.response.get("CancellationReasons", [])
            failed = [
                {"index": i, "code": r.get("Code"), "message": r.get("Message")}
                for i, r in enumerate(reasons)
                if r.get("Code") != "None"
            ]
            raise ValueError(
                f"Transaction cancelled. Reasons: {failed}"
            ) from e

        if code == "TransactionInProgressException":
            # Conflict with another in-progress transaction on the same item
            # SDK retries automatically, but if reached here, retries exhausted
            raise RuntimeError("Transaction conflict — retry after backoff") from e

        raise
```

#### CLI — Examples of conditional operations and transactions

```bash
# 1. PutItem with attribute_not_exists (put-if-not-exists)
aws dynamodb put-item \
  --table-name accounts-table \
  --item '{
    "PK": {"S": "ACCOUNT#acc-001"},
    "SK": {"S": "METADATA"},
    "balance": {"N": "1000"},
    "version": {"N": "0"},
    "status": {"S": "ACTIVE"}
  }' \
  --condition-expression "attribute_not_exists(PK)"

# 2. UpdateItem with optimistic locking (version counter)
aws dynamodb update-item \
  --table-name accounts-table \
  --key '{"PK": {"S": "ACCOUNT#acc-001"}, "SK": {"S": "METADATA"}}' \
  --update-expression "SET balance = balance - :amount, version = version + :inc" \
  --condition-expression "version = :expected AND balance >= :amount AND #s = :active" \
  --expression-attribute-names '{"#s": "status"}' \
  --expression-attribute-values '{
    ":amount": {"N": "200"},
    ":inc": {"N": "1"},
    ":expected": {"N": "0"},
    ":active": {"S": "ACTIVE"}
  }' \
  --return-values ALL_NEW

# 3. TransactWriteItems via CLI (simplified transfer)
aws dynamodb transact-write-items \
  --transact-items '[
    {
      "Update": {
        "TableName": "accounts-table",
        "Key": {"PK": {"S": "ACCOUNT#acc-001"}, "SK": {"S": "METADATA"}},
        "UpdateExpression": "SET balance = balance - :amt, version = version + :inc",
        "ConditionExpression": "balance >= :amt AND version = :v",
        "ExpressionAttributeValues": {
          ":amt": {"N": "100"},
          ":inc": {"N": "1"},
          ":v": {"N": "1"}
        }
      }
    },
    {
      "Update": {
        "TableName": "accounts-table",
        "Key": {"PK": {"S": "ACCOUNT#acc-002"}, "SK": {"S": "METADATA"}},
        "UpdateExpression": "SET balance = balance + :amt, version = version + :inc",
        "ExpressionAttributeValues": {
          ":amt": {"N": "100"},
          ":inc": {"N": "1"}
        }
      }
    }
  ]' \
  --client-request-token "transfer-$(uuidgen)" \
  --return-consumed-capacity TOTAL

# 4. TransactGetItems — atomic snapshot of multiple accounts
aws dynamodb transact-get-items \
  --transact-items '[
    {
      "Get": {
        "TableName": "accounts-table",
        "Key": {"PK": {"S": "ACCOUNT#acc-001"}, "SK": {"S": "METADATA"}},
        "ProjectionExpression": "balance, version"
      }
    },
    {
      "Get": {
        "TableName": "accounts-table",
        "Key": {"PK": {"S": "ACCOUNT#acc-002"}, "SK": {"S": "METADATA"}},
        "ProjectionExpression": "balance, version"
      }
    }
  ]'

# 5. Check transaction costs in CloudWatch
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB \
  --metric-name ConsumedWriteCapacityUnits \
  --dimensions Name=TableName,Value=accounts-table \
  --start-time "$(date -u -d '1 hour ago' '+%Y-%m-%dT%H:%M:%SZ' 2>/dev/null || date -u -v-1H '+%Y-%m-%dT%H:%M:%SZ')" \
  --end-time "$(date -u '+%Y-%m-%dT%H:%M:%SZ')" \
  --period 300 \
  --statistics Sum

# 6. Check transaction conflicts
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB \
  --metric-name TransactionConflict \
  --dimensions Name=TableName,Value=accounts-table \
  --start-time "$(date -u -d '1 hour ago' '+%Y-%m-%dT%H:%M:%SZ' 2>/dev/null || date -u -v-1H '+%Y-%m-%dT%H:%M:%SZ')" \
  --end-time "$(date -u '+%Y-%m-%dT%H:%M:%SZ')" \
  --period 300 \
  --statistics Sum
```


---

## Common pitfalls

**1. Same item twice in the same transaction → validation error**  
[FACT] `TransactWriteItems` does not allow two actions on the same item within the same call — neither `ConditionCheck` + `Update`, nor two `Update`s. If you need to verify and update the same item, combine the condition directly in the `Update` with `ConditionExpression`.

**2. Transactions in Global Tables: ACID only in the write region**  
[FACT] `TransactWriteItems` guarantees atomicity only in the region where it was invoked. Replication to other regions occurs asynchronously and may show partial state transitionally. Do not assume cross-region consistency with transactions.

**3. TransactionCanceledException: extract reasons per item**  
[FACT] When a transaction is cancelled, the exception contains `CancellationReasons` — an array with one entry per item, in the same order as `TransactItems`. Only items that failed have `Code` different from `"None"`. Without extracting these reasons, it is impossible to know which condition failed.

**4. `BatchWriteItem` is not transactional — common confusion**  
[FACT] `BatchWriteItem` is a convenience operation (reduces round-trips), not transactional. Some items may be written successfully while others fail. For all-or-nothing atomicity, use `TransactWriteItems`. The cost difference: `BatchWriteItem` = 1× WCU/item; `TransactWriteItems` = 2× WCU/item.

**5. Reserved words in ConditionExpression**  
[FACT] `status`, `name`, `size`, `value`, `type`, `timestamp`, among others, are reserved words in DynamoDB. Using them directly in expressions causes syntax errors. Always use `ExpressionAttributeNames` with the `#` prefix to substitute: `"#s": "status"`.

**6. Doubled cost when using transactions with DAX**  
[FACT] `TransactWriteItems` via DAX: beyond the normal 2× WCUs, DAX internally executes a `TransactGetItems` to populate the cache after each transactional write — adding 2× RCUs per item. A 500-byte item in `TransactWriteItems` via DAX consumes 2 WCUs + 2 RCUs.

---

## Reflection exercise

You are modeling a ticket reservation system. When a customer buys a ticket, you need to:
1. Verify the event still has available capacity (`availableSeats > 0`)
2. Decrement `availableSeats` on the event item
3. Create a ticket item for the customer
4. Record the sale in an audit item

Answer:

1. What is the problem with implementing this flow with 3 separate sequential `PutItem`/`UpdateItem` calls? Describe a concrete race condition scenario that can occur.

2. You decide to use `TransactWriteItems`. Which 3 or 4 actions do you put in the transaction and what `ConditionExpression` do you use to ensure the operation is safe? Write the skeleton of the call.

3. The system has 100 req/s of simultaneous purchases for the same event. How many `TransactionConflict` would you expect? What can be done to reduce conflicts while maintaining consistency?

---

## Resources for further study

- [FACT] DynamoDB Transactions — How it works: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/transaction-apis.html
- [FACT] ConditionExpression — CLI examples: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Expressions.ConditionExpressions.html
- [FACT] Optimistic locking with version number: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/BestPractices_OptimisticLocking.html
- [FACT] Best practices — implementing version control: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/BestPractices_ImplementingVersionControl.html

---
