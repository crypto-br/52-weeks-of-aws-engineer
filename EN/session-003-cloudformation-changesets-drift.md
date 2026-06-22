# Session 003 — CloudFormation: changesets, drift detection, and stack policies

**Estimated duration:** 60 minutes
**Prerequisites:** session-002 — CloudFormation: stacks, templates, parameters, outputs, Ref/GetAtt

---

## Objective

By the end, you will be able to create and review a changeset before applying it, detect drift in resources outside the CloudFormation lifecycle, and configure a stack policy to protect critical resources from accidental replacement.

---

## Context

[CONSENSUS] The main risk in IaC operations in production is not the initial deploy — it's the update. An `aws cloudformation deploy` in production can delete a database, replace a security group with all its rules, or change the name of an SQS queue (losing in-flight messages) if the engineer doesn't understand the update behavior of each resource type.

This session covers the three tools that exist to mitigate this risk: changesets (to see what will change before changing), drift detection (to know when the real world has diverged from the template), and stack policies (to prevent certain changes from happening even if requested).

[FACT] Changesets and stack policies are CloudFormation's own features. Drift detection is an asynchronous operation that can take minutes on large stacks and is not executed automatically — you need to trigger it explicitly.

---

## Core concepts

### 1. Update behaviors — what can happen to a resource during an update

Before understanding changesets, you need to understand what CloudFormation can do to a resource during an update. Each property of each resource type has a documented "update behavior".

[FACT] There are three categories:

```
Update with No Interruption
  → CloudFormation updates the resource without interrupting operation
  → The resource maintains its physical ID
  → E.g., changing tags on an EC2, changing a Security Group description

Update with Some Interruption
  → CloudFormation updates the resource with temporary interruption
  → The resource maintains its physical ID
  → E.g., changing the instance type of an EC2 (reboot required)

Replacement
  → CloudFormation creates a NEW resource, updates references,
    and deletes the old resource
  → The resource receives a NEW physical ID
  → E.g., changing an S3 bucket name, changing an RDS engine,
        changing a Security Group's VPC
```

**Why Replacement is the most dangerous:**

```
Previous state:                     After Replacement:
  AppBucket (bucket-prod-123)  →     AppBucket (bucket-prod-456) [new]
                                     bucket-prod-123 [deleted]

If DeletionPolicy = Delete:  all objects are lost
If DeletionPolicy = Retain:  old bucket remains orphaned in the account
```

[FACT] To identify the update behavior of a specific property, consult the resource documentation in the "Update requires" column for each property. There is no universal rule — it depends on the service and the property.

**Update behaviors diagram:**

```
Stack update
      │
      ├─ Property with "No Interruption"
      │         └─ Resource updated in-place, no impact
      │
      ├─ Property with "Some Interruption"
      │         └─ Resource restarted, minimal downtime
      │
      └─ Property with "Replacement"
                └─ New resource created → references updated → old one deleted
                   ⚠️ Physical ID changes, data may be lost
```

---

### 2. Changesets — see before doing

A changeset is an execution plan that CloudFormation calculates without applying anything. You create it, review it, and only then decide whether to execute.

**Complete flow via CLI:**

```bash
# 1. Create the changeset (applies nothing)
aws cloudformation create-change-set \
  --stack-name my-stack \
  --change-set-name my-changeset-$(date +%Y%m%d%H%M) \
  --template-body file://template.yaml \
  --parameters ParameterKey=Env,ParameterValue=prod \
  --capabilities CAPABILITY_NAMED_IAM

# 2. Wait for the changeset to be ready (status: CREATE_COMPLETE)
aws cloudformation wait change-set-create-complete \
  --stack-name my-stack \
  --change-set-name my-changeset-20260506

# 3. Review the changeset
aws cloudformation describe-change-set \
  --stack-name my-stack \
  --change-set-name my-changeset-20260506 \
  --query 'Changes[*].ResourceChange.[Action,LogicalResourceId,ResourceType,Replacement]' \
  --output table

# 4a. Execute (if approved)
aws cloudformation execute-change-set \
  --stack-name my-stack \
  --change-set-name my-changeset-20260506

# 4b. Cancel (if not approved)
aws cloudformation delete-change-set \
  --stack-name my-stack \
  --change-set-name my-changeset-20260506
```

**Reading the describe-change-set output:**

The most important field is `Changes[].ResourceChange`. Each item has:

```json
{
  "Action": "Modify",          // Add | Modify | Remove
  "LogicalResourceId": "AppDB",
  "ResourceType": "AWS::RDS::DBInstance",
  "Replacement": "True",       // True | False | Conditional
  "Scope": ["Properties"],
  "Details": [
    {
      "Target": {
        "Attribute": "Properties",
        "Name": "DBInstanceClass",
        "RequiresRecreation": "Always"  // Never | Conditional | Always
      },
      "ChangeSource": "DirectModification"
    }
  ]
}
```

**The `Replacement` field and its values:**

```
False        → no modified property requires replacement
True         → at least one property requires replacement (certain)
Conditional  → replacement depends on other conditions evaluated at runtime
               (e.g., when a property changes to a specific value)
```

[FACT] `Conditional` is the most treacherous — CloudFormation cannot determine at planning time whether replacement will occur. Treat `Conditional` as `True` when reviewing changesets in production.

**Comparing `aws cloudformation deploy` vs manual changeset:**

```
aws cloudformation deploy
  └── creates changeset internally
  └── executes automatically
  └── you don't see the changeset before execution
  └── --no-execute-changeset creates the changeset but does NOT execute

aws cloudformation create-change-set + execute-change-set
  └── explicit flow — you control each step
  └── recommended in production pipelines with human approval
```

**Multiple changesets on a stack:**

[FACT] A stack can have multiple pending changesets simultaneously, but only one can be executed — upon execution, CloudFormation automatically deletes all other changesets on the stack (they become obsolete after execution).

---

### 3. Drift detection — when the real world diverges from the template

Drift happens when someone modifies a resource outside of CloudFormation — via console, direct CLI, or another tool. The template says one thing, the resource is in a different state.

**Triggering drift detection:**

```bash
# Detect drift on the entire stack (asynchronous operation)
aws cloudformation detect-stack-drift \
  --stack-name my-stack
# Returns: { "StackDriftDetectionId": "abc-123" }

# Track the detection status
aws cloudformation describe-stack-drift-detection-status \
  --stack-drift-detection-id abc-123
# DetectionStatus: DETECTION_IN_PROGRESS | DETECTION_COMPLETE | DETECTION_FAILED

# View the result per resource
aws cloudformation describe-stack-resource-drifts \
  --stack-name my-stack \
  --stack-resource-drift-filter-status DRIFTED \
  --query 'StackResourceDrifts[*].[LogicalResourceId,ResourceType,StackResourceDriftStatus]' \
  --output table

# Detect drift on a specific resource
aws cloudformation detect-stack-resource-drift \
  --stack-name my-stack \
  --logical-resource-id AppBucket
```

**Drift status per resource:**

```
IN_SYNC     → properties match the template
DRIFTED     → at least one property differs from the template
NOT_CHECKED → resource doesn't support drift detection or wasn't checked
DELETED     → resource was deleted outside CloudFormation
```

**What drift detection checks:**

[FACT] CloudFormation only compares properties **explicitly defined in the template** with the current state of the resource. Properties not declared in the template (which retain service default values) are not checked.

```yaml
# Template declares only BucketName and VersioningConfiguration
AppBucket:
  Type: AWS::S3::Bucket
  Properties:
    BucketName: my-bucket
    VersioningConfiguration:
      Status: Enabled

# Someone added a lifecycle policy via console
# → drift detection WILL detect the lifecycle as "added" (wasn't in the template)
# → drift detection will NOT report the absence of CORS (was never declared)
```

**Important drift detection limitations — [FACT]:**

```
1. Not all resource types support drift detection
   (check the list at "Resources that support import and drift detection operations")

2. Detection is asynchronous and can take several minutes on large stacks

3. Drift detection does NOT fix drift — it only reports it
   To fix: either manually update the resource to match the template state,
   or update the template to reflect the current state

4. There is no native continuous drift detection — you need to schedule via EventBridge
   or use AWS Config with the `cloudformation-stack-drift-detection-check` rule

5. Stacks with REVIEW_IN_PROGRESS status cannot have drift detected
```

**Drift lifecycle diagram:**

```
Template                     Actual Resource
─────────                    ────────────────
VersioningConfiguration: Enabled
                             VersioningConfiguration: Enabled   ← IN_SYNC

[Someone goes to the console and disables versioning]

VersioningConfiguration: Enabled
                             VersioningConfiguration: Suspended ← DRIFTED

Options to resolve:
  A) Re-enable versioning in the console/CLI → IN_SYNC
  B) Update the template to Suspended and deploy → IN_SYNC
  C) Deploy the original template (without changes) → CloudFormation
     detects the difference and re-enables versioning as part of the update
```

---

### 4. Stack policies — declarative protection against destructive updates

A stack policy is a JSON document that defines which update actions are allowed on which resources. Once defined, every stack update passes through the policy before being executed.

**Policy structure:**

```json
{
  "Statement": [
    {
      "Effect": "Allow" | "Deny",
      "Principal": "*",
      "Action": [ "Update:Modify", "Update:Replace", "Update:Delete", "Update:*" ],
      "Resource": "LogicalResourceId/ResourceName" | "*",
      "Condition": { ... }   // optional
    }
  ]
}
```

**The four possible Actions:**

```
Update:Modify    → update that does not replace or delete
Update:Replace   → update that replaces the resource (Replacement = True)
Update:Delete    → removal of the resource from the template
Update:*         → any update (wildcard)
```

**Default behavior — [FACT]:**

Without stack policy: all resources can be updated without restriction.
With stack policy: all resources are protected by default (`Deny Update:*`). You must explicitly declare what is allowed.

**Practical example — protect RDS and SQS, allow the rest:**

```json
{
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "Update:*",
      "Resource": "*"
    },
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": ["Update:Replace", "Update:Delete"],
      "Resource": "LogicalResourceId/ProductionDatabase"
    },
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": ["Update:Replace", "Update:Delete"],
      "Resource": "LogicalResourceId/OrdersQueue"
    }
  ]
}
```

[FACT] Deny always takes precedence over Allow when there is statement overlap. The logic is identical to IAM.

**Applying and managing the policy:**

```bash
# Apply the policy when creating the stack
aws cloudformation create-stack \
  --stack-name my-stack \
  --template-body file://template.yaml \
  --stack-policy-body file://stack-policy.json

# Apply to an existing stack
aws cloudformation set-stack-policy \
  --stack-name my-stack \
  --stack-policy-body file://stack-policy.json

# View the current policy
aws cloudformation get-stack-policy \
  --stack-name my-stack

# Temporary override to update a protected resource
# (the original policy is automatically restored after the update)
aws cloudformation update-stack \
  --stack-name my-stack \
  --template-body file://template.yaml \
  --stack-policy-during-update-body file://override-policy.json
```

**Temporary override — emergency policy:**

```json
// override-policy.json — allows replacement only for RDS
{
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "Update:*",
      "Resource": "LogicalResourceId/ProductionDatabase"
    }
  ]
}
```

[FACT] The override via `--stack-policy-during-update-body` is applied only during that specific update. After the update completes (success or rollback), the original policy is automatically restored.

---

### 5. Stack termination protection — complement to stack policies

Stack policy protects resources from updates. Termination protection protects the entire stack from being deleted.

```bash
# Enable
aws cloudformation update-termination-protection \
  --stack-name my-stack \
  --enable-termination-protection

# Verify
aws cloudformation describe-stacks \
  --stack-name my-stack \
  --query 'Stacks[0].EnableTerminationProtection'

# Disable (required before deleting)
aws cloudformation update-termination-protection \
  --stack-name my-stack \
  --no-enable-termination-protection
```

**Difference between protections:**

```
Termination protection  → prevents DELETE of the entire stack
Stack policy            → prevents destructive UPDATE of specific resources
DeletionPolicy: Retain  → resource behavior when the stack is deleted
```

All three are complementary and independent.

---

## Practical example

**Scenario:** you have a production stack with an RDS and need to update the instance type — a `Some Interruption` operation. You want to ensure you won't accidentally replace the database.

```bash
# 1. View the current state of the stack
aws cloudformation describe-stacks \
  --stack-name prod-stack \
  --query 'Stacks[0].[StackStatus,EnableTerminationProtection]'

# 2. Check for drift before any operation
aws cloudformation detect-stack-drift --stack-name prod-stack
# wait...
aws cloudformation describe-stack-resource-drifts \
  --stack-name prod-stack \
  --stack-resource-drift-filter-status DRIFTED \
  --output table
# If there is drift, evaluate before proceeding

# 3. Create changeset with the instance class change
aws cloudformation create-change-set \
  --stack-name prod-stack \
  --change-set-name update-rds-class-$(date +%Y%m%d) \
  --template-body file://template.yaml \
  --parameters \
    ParameterKey=DBInstanceClass,ParameterValue=db.t3.large \
  --capabilities CAPABILITY_NAMED_IAM

aws cloudformation wait change-set-create-complete \
  --stack-name prod-stack \
  --change-set-name update-rds-class-20260506

# 4. Review: verify that Replacement is False (expected for class change)
aws cloudformation describe-change-set \
  --stack-name prod-stack \
  --change-set-name update-rds-class-20260506 \
  --query 'Changes[*].ResourceChange.[Action,LogicalResourceId,Replacement,Scope]' \
  --output table

# Expected output:
# Action   | LogicalResourceId | Replacement | Scope
# Modify   | ProductionDB      | False       | ['Properties']
# ✅ Replacement = False → safe to proceed

# 5. Execute after confirmation
aws cloudformation execute-change-set \
  --stack-name prod-stack \
  --change-set-name update-rds-class-20260506

# 6. Track events in real time
aws cloudformation describe-stack-events \
  --stack-name prod-stack \
  --query 'StackEvents[0:10].[Timestamp,LogicalResourceId,ResourceStatus,ResourceStatusReason]' \
  --output table
```

---

## Common pitfalls

**1. Trusting `Replacement: False` without checking `RequiresRecreation` in Details**

The `Replacement` field at the resource change level is an aggregate. To understand which specific property is causing the change — and whether it could cause replacement in different scenarios — you need to look at `Changes[].ResourceChange.Details[].Target.RequiresRecreation`. A changeset may show `Replacement: False` today and `Replacement: True` tomorrow if the property value changes.

**2. Thinking drift detection covers all resources**

Several resource types do not support drift detection — among them some Lambda resources, resources from newer services, and custom resources (`AWS::CloudFormation::CustomResource`). Before trusting a clean drift report, check which resources in the stack are marked as `NOT_CHECKED` in the output.

**3. Stack policy blocking its own rollback**

[FACT] If a stack policy denies `Update:Replace` on a resource, and an update fails and tries to roll back by creating a new version of the resource, the rollback can also be blocked by the policy — leaving the stack in `UPDATE_ROLLBACK_FAILED`. To escape this state: `aws cloudformation continue-update-rollback` with `--resources-to-skip` to skip the problematic resource.

---

## Reflection exercise

You maintain a production stack with the following resources: an RDS PostgreSQL, an SQS queue, an S3 bucket, and an ALB. A colleague creates a stack policy that only denies `Update:Replace` and `Update:Delete` for the RDS and the SQS queue, and allows `Update:*` for everything else.

Three months later, someone changes the S3 `BucketName` in the template (a Replacement operation) and deploys. The bucket is replaced and data is lost.

Where did the protection fail? Rewrite the stack policy to cover this case without blocking legitimate maintenance operations on the other resources. Also consider: what would need to be done in the template beyond the stack policy to ensure protection in multiple layers?

---

## Resources for deeper learning

**Changesets:**
- [Update CloudFormation stacks using change sets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-changesets.html) — covers the complete cycle: creation, viewing, and execution, with the output fields explained.

**Update behaviors:**
- [Understand update behaviors of stack resources](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-update-behaviors.html) — reference of the three behaviors with examples of resources and properties for each category.

**Drift detection:**
- [Detect unmanaged configuration changes to stacks and resources with drift detection](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-stack-drift.html) — includes the list of supported resources and the detailed format of drift output per resource.

**Stack policies:**
- [Prevent updates to stack resources](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/protect-stack-resources.html) — JSON policy structure, the four available actions, and how to do temporary override.

---
