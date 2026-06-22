# Session 043 — AWS Budgets with Automatic Actions and Tagging Strategy for Chargeback

**Estimated duration:** 60 min  
**Prerequisite:** session-042 (Cost Explorer, Anomaly Detection, Compute Optimizer)

---

## Session Objectives

- Configure AWS Budgets with automatic actions (IAM policy, SCP, EC2/RDS instances)
- Understand the AUTOMATIC vs MANUAL execution model and when to use each
- Design a mandatory tag hierarchy for chargeback by project/team
- Distinguish the three cost allocation models (account-based, team-based, tag-based)
- Use AWS Config Rules to detect resources non-compliant with the tagging policy
- Use Tag Policies for value normalization in Organizations

---

## 1. AWS Budgets — Types and Structure

[FACT] AWS Budgets supports six budget types:

```
╔══════════════════════════════════╦═══════════════════════════════════════════╗
║ Type                             ║ What it monitors                          ║
╠══════════════════════════════════╬═══════════════════════════════════════════╣
║ COST                             ║ Monetary cost ($)                         ║
║ USAGE                            ║ Usage volume (GB, hours, etc.)            ║
║ RI_UTILIZATION                   ║ % usage of Reserved Instances             ║
║ RI_COVERAGE                      ║ % On-Demand hours covered by RI           ║
║ SAVINGS_PLANS_UTILIZATION        ║ % usage of purchased Savings Plans        ║
║ SAVINGS_PLANS_COVERAGE           ║ % On-Demand hours covered by SP           ║
╚══════════════════════════════════╩═══════════════════════════════════════════╝
```

[FACT] Budget alerts have two threshold types:
- `ACTUAL`: cost/usage already incurred (retroactive — does not prevent, only notifies)
- `FORECASTED`: projection from the forecasting model for the current period

[FACT] Default limit: **20,000 budgets per account**. Cost: the first 2 budgets are free; from the 3rd onward, $0.02/day per budget (≈ $0.60/month).

---

## 2. Budget Actions — Automatic Actions

[FACT] When a threshold is reached, Budget Actions can automatically execute (or wait for manual approval) one of the following actions:

```
╔═══════════════════════════════════╦═════════════════════════════════════════════╗
║ Action Type                       ║ What it does                                ║
╠═══════════════════════════════════╬═════════════════════════════════════════════╣
║ APPLY_IAM_POLICY                  ║ Attaches an IAM policy (deny) to user/role/ ║
║                                   ║ group — prevents provisioning new resources  ║
╠═══════════════════════════════════╬═════════════════════════════════════════════╣
║ APPLY_SCP_POLICY                  ║ Applies an SCP to a member account in the   ║
║                                   ║ Organization — only management account can   ║
╠═══════════════════════════════════╬═════════════════════════════════════════════╣
║ EC2_STOP_INSTANCES /              ║ Stops or terminates specific EC2 instances  ║
║ EC2_TERMINATE_INSTANCES           ║ in the target account                       ║
╠═══════════════════════════════════╬═════════════════════════════════════════════╣
║ RDS_STOP_INSTANCES /              ║ Stops specific RDS instances                ║
║ RDS_TERMINATE_INSTANCES           ║ (cannot target another account)             ║
╚═══════════════════════════════════╩═════════════════════════════════════════════╝
```

### 2.1 AUTOMATIC vs MANUAL

[FACT] Each configured action can be:
- **AUTOMATIC** (`ApprovalModel="AUTOMATIC"`): executes immediately when the threshold is reached
- **MANUAL** (`ApprovalModel="MANUAL"`): places the action in `STANDBY` state — waits for approval in the console or via `execute-budget-action`

[CONSENSUS] **Stop/terminate instance** actions should preferably be `MANUAL` in production — the risk of accidentally terminating production instances is high. For dev/sandbox environments, `AUTOMATIC` is appropriate.

[FACT] The applied action **is not automatically reverted** when the cost drops below the threshold. The operator must manually remove the SCP or detach the IAM policy.

### 2.2 Required IAM Role

[FACT] AWS Budgets needs an IAM Role with specific permissions to execute actions. AWS provides the managed policy `AWSBudgetsActionsRolePolicyForResourceAdministrationWithSSM`:

```json
// Trust policy required for Budget Actions
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "budgets.amazonaws.com" },
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": { "aws:SourceAccount": "ACCOUNT_ID" }
    }
  }]
}

// Minimum permissions for SCP + IAM actions
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "iam:AttachGroupPolicy", "iam:AttachRolePolicy", "iam:AttachUserPolicy",
        "iam:DetachGroupPolicy", "iam:DetachRolePolicy", "iam:DetachUserPolicy"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": ["organizations:AttachPolicy", "organizations:DetachPolicy"],
      "Resource": "*"
    }
  ]
}
```

---

## 3. CDK Python — Budget with Automatic SCP Action

```python
from aws_cdk import (
    Stack, aws_budgets as budgets, aws_iam as iam,
    aws_sns as sns, aws_organizations as organizations,
)
from constructs import Construct

class BudgetActionsStack(Stack):
    def __init__(self, scope: Construct, construct_id: str,
                 target_account_id: str,
                 **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # ──────────────────────────────────────────────────────────────
        # IAM Role for Budget Actions
        # ──────────────────────────────────────────────────────────────
        budget_actions_role = iam.Role(self, "BudgetActionsRole",
            assumed_by=iam.ServicePrincipal("budgets.amazonaws.com",
                conditions={
                    "StringEquals": {
                        "aws:SourceAccount": self.account
                    }
                }
            ),
            managed_policies=[
                iam.ManagedPolicy.from_aws_managed_policy_name(
                    "AWSBudgetsActionsRolePolicyForResourceAdministrationWithSSM"
                )
            ],
        )

        # ──────────────────────────────────────────────────────────────
        # Deny SCP — denies provisioning of new EC2 resources
        # Will be attached to target account when budget reaches 80%
        # ──────────────────────────────────────────────────────────────
        deny_ec2_scp = organizations.CfnPolicy(self, "DenyEC2ProvisionSCP",
            name="DenyNewEC2Provision",
            description="Applied automatically when budget reaches 80% — blocks new EC2 launches",
            type="SERVICE_CONTROL_POLICY",
            content={
                "Version": "2012-10-17",
                "Statement": [{
                    "Sid": "DenyEC2Launch",
                    "Effect": "Deny",
                    "Action": [
                        "ec2:RunInstances",
                        "ec2:StartInstances",
                    ],
                    "Resource": "*",
                    "Condition": {
                        "StringNotLike": {
                            "aws:PrincipalArn": [
                                # Exceptions: automation and SRE accounts
                                f"arn:aws:iam::{target_account_id}:role/AutomationRole",
                                f"arn:aws:iam::{target_account_id}:role/SREBreakGlass",
                            ]
                        }
                    }
                }]
            },
            target_ids=[target_account_id],  # OU ID or Account ID
        )

        # SNS for notifications
        alert_topic = sns.Topic(self, "BudgetAlertTopic")

        # ──────────────────────────────────────────────────────────────
        # Budget with multiple thresholds and chained actions
        # ──────────────────────────────────────────────────────────────
        budget = budgets.CfnBudget(self, "ProjectBudget",
            budget=budgets.CfnBudget.BudgetDataProperty(
                budget_name="checkout-service-prod-monthly",
                budget_type="COST",
                time_unit="MONTHLY",
                budget_limit=budgets.CfnBudget.SpendProperty(
                    amount=10000, unit="USD"
                ),
                cost_filters={
                    "TagKeyValue": ["user:project$checkout-service"]
                },
                cost_types=budgets.CfnBudget.CostTypesProperty(
                    include_tax=True,
                    include_subscription=True,
                    use_amortized=True,   # AmortizedCost — includes amortized RI/SP
                    use_blended=False,
                ),
            ),
            notifications_with_subscribers=[
                # Threshold 1: 80% actual → notification
                budgets.CfnBudget.NotificationWithSubscribersProperty(
                    notification=budgets.CfnBudget.NotificationProperty(
                        comparison_operator="GREATER_THAN",
                        notification_type="ACTUAL",
                        threshold=80,
                        threshold_type="PERCENTAGE",
                    ),
                    subscribers=[
                        budgets.CfnBudget.SubscriberProperty(
                            address=alert_topic.topic_arn,
                            subscription_type="SNS",
                        )
                    ],
                ),
                # Threshold 2: 100% forecasted → notification
                budgets.CfnBudget.NotificationWithSubscribersProperty(
                    notification=budgets.CfnBudget.NotificationProperty(
                        comparison_operator="GREATER_THAN",
                        notification_type="FORECASTED",
                        threshold=100,
                        threshold_type="PERCENTAGE",
                    ),
                    subscribers=[
                        budgets.CfnBudget.SubscriberProperty(
                            address="finops-team@company.com",
                            subscription_type="EMAIL",
                        )
                    ],
                ),
            ],
        )

        # ──────────────────────────────────────────────────────────────
        # Budget Action: at 80% actual → apply SCP (AUTOMATIC)
        # The action is separate from CfnBudget — uses CfnBudgetsAction
        # ──────────────────────────────────────────────────────────────
        # Note: CfnBudgetsAction is not directly supported by CDK L1
        # as a construct. Use aws_cdk.aws_budgets.CfnBudgetsAction
        from aws_cdk import aws_budgets as budgets_l1
        
        budgets_l1.CfnBudgetsAction(self, "ApplySCPAt80Pct",
            budget_name=budget.ref,  # reference to budget above
            action_type="APPLY_SCP_POLICY",
            action_threshold=budgets_l1.CfnBudgetsAction.ActionThresholdProperty(
                type="PERCENTAGE",
                value=80,
            ),
            execution_role_arn=budget_actions_role.role_arn,
            notification_type="ACTUAL",
            approval_model="AUTOMATIC",  # executes without manual approval
            definition=budgets_l1.CfnBudgetsAction.DefinitionProperty(
                scp_action_definition=budgets_l1.CfnBudgetsAction.ScpActionDefinitionProperty(
                    policy_id=deny_ec2_scp.ref,  # ID of the SCP created above
                    target_ids=[target_account_id],
                )
            ),
            subscribers=[
                budgets_l1.CfnBudgetsAction.SubscriberProperty(
                    type="SNS",
                    address=alert_topic.topic_arn,
                )
            ],
        )

        # Budget Action: 90% actual → stop staging instances (MANUAL — requires approval)
        budgets_l1.CfnBudgetsAction(self, "StopStagingAt90Pct",
            budget_name=budget.ref,
            action_type="EC2_STOP_INSTANCES",
            action_threshold=budgets_l1.CfnBudgetsAction.ActionThresholdProperty(
                type="PERCENTAGE",
                value=90,
            ),
            execution_role_arn=budget_actions_role.role_arn,
            notification_type="ACTUAL",
            approval_model="MANUAL",    # places in STANDBY — awaits approval
            definition=budgets_l1.CfnBudgetsAction.DefinitionProperty(
                ec2_action_definition=budgets_l1.CfnBudgetsAction.Ec2ActionDefinitionProperty(
                    instance_ids=["i-0staging123", "i-0staging456"],
                    region="us-east-1",
                )
            ),
            subscribers=[
                budgets_l1.CfnBudgetsAction.SubscriberProperty(
                    type="EMAIL",
                    address="oncall@company.com",
                )
            ],
        )
```

---

## 4. Cost Allocation Models

[FACT] The official whitepaper "Best Practices for Tagging AWS Resources" describes three models:

### 4.1 Account-based (least effort)

```
AWS Organizations:
  Management Account
    ├── OU: Workloads
    │     ├── Account: checkout-service (prod)
    │     ├── Account: checkout-service (staging)
    │     └── Account: payments-api (prod)
    └── OU: Shared Services
          └── Account: observability (shared)

Advantage: cost already isolated per account — no dependency on tags
Limitation: OU/Organization tags are NOT usable in Cost Explorer
```

[FACT] Tags on Organizational Units (OUs) in AWS Organizations **are not propagated** for cost allocation. To group accounts into cost categories, use **AWS Cost Categories**.

### 4.2 Business Unit / Team-based (moderate effort)

[FACT] Combine accounts by team/application and use **AWS Cost Categories** to aggregate:

```
Cost Category: "Checkout Platform"
  Rule 1: LinkedAccount IN [account-checkout-prod, account-checkout-staging]
  Rule 2: Tag project = "checkout-service"  (for resources in shared account)
```

### 4.3 Tag-based (most effort, most granular)

[FACT] When multiple teams share the same AWS account, tags are the only cost allocation mechanism per project. Requires rigor in application and **activation in the Billing console** of the management account.


---

## 5. Mandatory Tag Hierarchy

[CONSENSUS] The tagging whitepaper recommends categorizing tags into 4 groups, each with a distinct purpose:

```
╔════════════════╦═════════════════════╦══════════════════════════════════════╗
║ Category       ║ Tag Key             ║ Purpose / Examples                    ║
╠════════════════╬═════════════════════╬══════════════════════════════════════╣
║ TECHNICAL      ║ Application         ║ App name (e.g.: checkout-service)    ║
║                ║ Environment         ║ prod | staging | dev | sandbox       ║
║                ║ Version             ║ v2.3.1 (for release tracking)        ║
║                ║ Component           ║ api | worker | database | cache      ║
╠════════════════╬═════════════════════╬══════════════════════════════════════╣
║ BUSINESS       ║ CostCenter          ║ Accounting cost center code          ║
║                ║ Owner               ║ Technical owner's email              ║
║                ║ Project             ║ Business initiative (e.g.: pix-v2)   ║
║                ║ Team                ║ Engineering team name                ║
╠════════════════╬═════════════════════╬══════════════════════════════════════╣
║ SECURITY       ║ Confidentiality     ║ public | internal | confidential     ║
║                ║ Compliance          ║ pci | hipaa | lgpd | none            ║
╠════════════════╬═════════════════════╬══════════════════════════════════════╣
║ AUTOMATION     ║ Schedule            ║ "Mon-Fri:08:00-20:00" (auto stop)    ║
║                ║ BackupPolicy        ║ daily | weekly | none                ║
║                ║ PatchGroup          ║ SSM patch group                      ║
╚════════════════╩═════════════════════╩══════════════════════════════════════╝
```

[OPINION] Keeping business and technical tags separate facilitates delegation: technical teams manage `Application`/`Version`/`Component`; FinOps manages `CostCenter`/`Project`; security manages `Confidentiality`/`Compliance`.

### 5.1 Minimum Tags for Chargeback

[CONSENSUS] The minimum functional set for effective chargeback (based on the AWS whitepaper):

```
MANDATORY (without these, resource is non-compliant):
  • project      — primary cost allocation
  • environment  — separate prod from non-prod
  • owner        — responsible person's email

RECOMMENDED (for granularity):
  • cost-center  — accounting code for internal billing
  • team         — team grouping

DO NOT USE as Cost Allocation Tags:
  • Version / Component — high cardinality → excessive indexing cost
    (each unique tag combination generates a line in the billing report)
```

---

## 6. Enforcing Tags — Compliance Mechanisms

[FACT] AWS offers four complementary mechanisms for enforcing tagging:

### 6.1 AWS Config Rules — detect non-compliance

[FACT] The managed rule `required-tags` verifies whether resources have the mandatory tags. Supports up to **6 key:value pairs** per rule.

```python
# CDK Python: AWS Config Rule for mandatory tags
from aws_cdk import aws_config as config

required_tags_rule = config.ManagedRule(self, "RequiredTagsRule",
    identifier="REQUIRED_TAGS",
    input_parameters={
        "tag1Key": "project",
        "tag2Key": "environment",
        "tag2Value": "prod,staging,dev,sandbox",  # allowed values
        "tag3Key": "owner",
    },
    description="Verifies that EC2, RDS, and Lambda resources have mandatory tags",
)
```

### 6.2 Tag Policies (AWS Organizations) — normalize values

[FACT] Tag Policies (not to be confused with IAM policies) enforce consistent tag value formatting. Examples: `Environment` always in lowercase, values restricted to an allowed list.

```json
// Tag Policy — normalizes the "environment" tag
{
  "tags": {
    "environment": {
      "tag_key": {
        "@@assign": "environment"
      },
      "tag_value": {
        "@@assign": ["prod", "staging", "dev", "sandbox"]
      },
      "enforced_for": {
        "@@assign": [
          "ec2:instance",
          "rds:db",
          "lambda:function",
          "s3:bucket"
        ]
      }
    }
  }
}
```

[FACT] Tag Policies **do not block** resource creation without the tag — they only report non-compliance. To block, use an SCP combined with `aws:RequestedRegion` and `aws:TagKeys` condition.

### 6.3 SCP with tag condition — block creation without tag

```json
// SCP: blocks RunInstances without "project" tag
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "RequireProjectTagOnEC2",
    "Effect": "Deny",
    "Action": ["ec2:RunInstances"],
    "Resource": "arn:aws:ec2:*:*:instance/*",
    "Condition": {
      "Null": { "aws:RequestTag/project": "true" }
    }
  }]
}
```

[FACT] SCPs with `aws:RequestTag` condition block at creation time — they do not correct existing resources. To remediate retroactively, use Resource Groups Tag Editor or AWS CLI.

---

## 7. Python — Tag Compliance Scan with Resource Groups Tagging API

```python
import boto3
from collections import defaultdict
from dataclasses import dataclass, field

tagging = boto3.client("resourcegroupstaggingapi", region_name="us-east-1")
ec2 = boto3.client("ec2", region_name="us-east-1")

MANDATORY_TAGS = {"project", "environment", "owner"}
ALLOWED_ENVIRONMENTS = {"prod", "staging", "dev", "sandbox"}

@dataclass
class NonCompliantResource:
    arn: str
    resource_type: str
    missing_tags: set[str]
    invalid_tags: dict[str, str]  # tag_key → invalid_value
    existing_tags: dict[str, str]


def scan_tag_compliance(
    resource_type_filters: list[str] | None = None,
    tag_filters: list[dict] | None = None,
) -> list[NonCompliantResource]:
    """
    Scans all resources and returns those non-compliant with the tag policy.
    resource_type_filters: e.g. ["ec2:instance", "rds:db", "lambda:function"]
    """
    params: dict = {}
    if resource_type_filters:
        params["ResourceTypeFilters"] = resource_type_filters
    if tag_filters:
        params["TagFilters"] = tag_filters

    non_compliant = []
    paginator = tagging.get_paginator("get_resources")

    for page in paginator.paginate(**params):
        for resource in page.get("ResourceTagMappingList", []):
            arn = resource["ResourceARN"]
            tags = {t["Key"]: t["Value"] for t in resource.get("Tags", [])}
            tag_keys_lower = {k.lower() for k in tags}

            missing = set()
            invalid = {}

            # Check mandatory tags
            for required in MANDATORY_TAGS:
                if required not in tags:
                    missing.add(required)

            # Check allowed values for "environment"
            if "environment" in tags:
                env_val = tags["environment"].lower()
                if env_val not in ALLOWED_ENVIRONMENTS:
                    invalid["environment"] = tags["environment"]

            # Extract resource type from ARN (e.g.: arn:aws:ec2:...:instance/...)
            parts = arn.split(":")
            resource_type = f"{parts[2]}:{parts[5].split('/')[0]}" if len(parts) > 5 else arn

            if missing or invalid:
                non_compliant.append(NonCompliantResource(
                    arn=arn,
                    resource_type=resource_type,
                    missing_tags=missing,
                    invalid_tags=invalid,
                    existing_tags=tags,
                ))

    return sorted(non_compliant, key=lambda r: len(r.missing_tags), reverse=True)


def print_compliance_summary(resources: list[NonCompliantResource]):
    """Prints compliance report grouped by resource type."""
    by_type = defaultdict(list)
    for r in resources:
        by_type[r.resource_type].append(r)

    total_missing_by_tag: dict[str, int] = defaultdict(int)
    for r in resources:
        for t in r.missing_tags:
            total_missing_by_tag[t] += 1

    print(f"\n{'='*60}")
    print(f"TAG COMPLIANCE REPORT")
    print(f"Total non-compliant: {len(resources)}")
    print(f"\nMost missing tags:")
    for tag, count in sorted(total_missing_by_tag.items(), key=lambda x: -x[1]):
        print(f"  {tag:20} missing in {count:>4} resources")

    print(f"\nBy resource type:")
    for resource_type, res_list in sorted(by_type.items()):
        print(f"\n  {resource_type} ({len(res_list)} non-compliant):")
        for r in res_list[:5]:  # show up to 5 per type
            missing_str = ", ".join(sorted(r.missing_tags)) if r.missing_tags else "-"
            invalid_str = str(r.invalid_tags) if r.invalid_tags else "-"
            print(f"    {r.arn[-50:]:50} | missing: {missing_str} | invalid: {invalid_str}")
        if len(res_list) > 5:
            print(f"    ... and {len(res_list) - 5} more resources")


def apply_default_tags_for_project(
    resource_arns: list[str],
    project: str,
    environment: str,
    owner_email: str,
) -> dict:
    """
    Applies default tags to a list of resources.
    Use with caution — overwrites existing tags if keys match.
    """
    response = tagging.tag_resources(
        ResourceARNList=resource_arns,
        Tags={
            "project": project,
            "environment": environment,
            "owner": owner_email,
        },
    )
    failed = response.get("FailedResourcesMap", {})
    succeeded = len(resource_arns) - len(failed)
    return {
        "succeeded": succeeded,
        "failed": len(failed),
        "failed_details": failed,
    }


# Usage example
if __name__ == "__main__":
    print("Starting compliance scan...")
    non_compliant = scan_tag_compliance(
        resource_type_filters=["ec2:instance", "rds:db", "lambda:function", "s3:bucket"]
    )
    print_compliance_summary(non_compliant)

    # Export ARNs without "project" tag for remediation
    no_project_arns = [
        r.arn for r in non_compliant if "project" in r.missing_tags
    ]
    print(f"\nARNs without 'project' tag for remediation: {len(no_project_arns)}")
```

---

## 8. CLI — Essential Examples

```bash
# ── BUDGETS ────────────────────────────────────────────────────────────────

# 1. Create simple budget via JSON file (for complex budgets)
cat > /tmp/budget.json << 'EOF'
{
  "BudgetName": "checkout-service-monthly",
  "BudgetLimit": {"Amount": "10000", "Unit": "USD"},
  "CostFilters": {"TagKeyValue": ["user:project$checkout-service"]},
  "TimeUnit": "MONTHLY",
  "BudgetType": "COST",
  "CostTypes": {
    "IncludeTax": true,
    "IncludeSubscription": true,
    "UseAmortized": true,
    "UseBlended": false
  }
}
EOF

aws budgets create-budget \
  --account-id $(aws sts get-caller-identity --query Account --output text) \
  --budget file:///tmp/budget.json

# 2. List budgets in the account
aws budgets describe-budgets \
  --account-id $(aws sts get-caller-identity --query Account --output text) \
  --query 'Budgets[*].{Name:BudgetName,Type:BudgetType,Limit:BudgetLimit.Amount,Actual:CalculatedSpend.ActualSpend.Amount,Forecast:CalculatedSpend.ForecastedSpend.Amount}' \
  --output table

# 3. List actions for a budget
aws budgets describe-budget-actions-for-budget \
  --account-id $(aws sts get-caller-identity --query Account --output text) \
  --budget-name "checkout-service-monthly" \
  --query 'Actions[*].{ID:ActionId,Type:ActionType,Status:Status,Approval:ApprovalModel}'

# 4. List all actions in the account (all budgets)
aws budgets describe-budget-actions-for-account \
  --account-id $(aws sts get-caller-identity --query Account --output text) \
  --query 'Actions[?Status==`STANDBY`].[BudgetName,ActionId,ActionType]' \
  --output table

# 5. Manually execute an action in STANDBY
aws budgets execute-budget-action \
  --account-id $(aws sts get-caller-identity --query Account --output text) \
  --budget-name "checkout-service-monthly" \
  --action-id "abc12345-def6-7890-ghij-klmnopqrstuv" \
  --execution-type APPROVE_BUDGET_ACTION

# 6. Reverse an executed action (remove SCP or IAM policy)
aws budgets execute-budget-action \
  --account-id $(aws sts get-caller-identity --query Account --output text) \
  --budget-name "checkout-service-monthly" \
  --action-id "abc12345-def6-7890-ghij-klmnopqrstuv" \
  --execution-type REVERSE_BUDGET_ACTION


# ── TAGGING AND COMPLIANCE ─────────────────────────────────────────────────

# 7. List EC2 resources WITHOUT the "project" tag (untagged instances)
aws ec2 describe-instances \
  --filters "Name=tag-key,Values=project" \
  --query 'Reservations[*].Instances[*].InstanceId' \
  --output text > /tmp/with-project-tag.txt

aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].InstanceId' \
  --output text > /tmp/all-instances.txt

# Difference = instances without tag
comm -23 <(sort /tmp/all-instances.txt) <(sort /tmp/with-project-tag.txt)

# 8. Resource Groups Tagging API — list resources without mandatory tag
aws resourcegroupstaggingapi get-resources \
  --resource-type-filters "ec2:instance" "lambda:function" "rds:db" \
  --tags-per-page 100 \
  --query 'ResourceTagMappingList[?!contains(Tags[*].Key, `project`)].[ResourceARN]' \
  --output text

# 9. Apply tags in batch via Tag Editor API
aws resourcegroupstaggingapi tag-resources \
  --resource-arn-list \
    "arn:aws:ec2:us-east-1:123456789012:instance/i-0abc123" \
    "arn:aws:lambda:us-east-1:123456789012:function:my-function" \
  --tags project=checkout-service,environment=prod,owner=team@company.com

# 10. AWS Config compliance report — resources without tags
aws configservice get-compliance-details-by-config-rule \
  --config-rule-name "required-tags" \
  --compliance-types "NON_COMPLIANT" \
  --query 'EvaluationResults[*].EvaluationResultIdentifier.EvaluationResultQualifier.ResourceId' \
  --output text

# 11. Tag Policies — list tag policies in the Organization
aws organizations list-policies \
  --filter TAG_POLICY \
  --query 'Policies[*].{Name:Name,Id:Id,Arn:Arn}'

# 12. Check effective tag policy for an account
aws organizations describe-effective-policy \
  --policy-type TAG_POLICY \
  --target-id 123456789012   # account ID

# 13. Cost Categories — create category to group accounts by BU
aws ce create-cost-category-definition \
  --name "BusinessUnit" \
  --rule-version "CostCategoryExpression.v1" \
  --rules '[
    {
      "Value": "Checkout Platform",
      "Rule": {
        "Or": [
          {"Dimensions": {"Key": "LINKED_ACCOUNT", "Values": ["111111111111", "222222222222"]}},
          {"Tags": {"Key": "project", "Values": ["checkout-service", "payments-api"]}}
        ]
      }
    },
    {
      "Value": "Shared Services",
      "Rule": {
        "Dimensions": {"Key": "LINKED_ACCOUNT", "Values": ["333333333333"]}
      }
    }
  ]' \
  --default-value "Unallocated"
```

---

## 9. Showback vs Chargeback

[FACT] According to the AWS whitepaper:

```
SHOWBACK
  Purpose:   visibility — "you generated this cost"
  Result:    awareness, behavior change
  Impact:    no real financial transfer
  Tooling:   Cost Explorer + dashboards

CHARGEBACK
  Purpose:   recovery — "you pay for this cost"
  Result:    direct financial responsibility
  Impact:    cost transfer to team's cost center
  Tooling:   CUR (Cost and Usage Report) + finance system
```

[CONSENSUS] AWS recommends starting with Showback to create cost awareness before implementing Chargeback — the cultural change is necessary for Chargeback to work properly.

### 9.1 Unallocatable Spend

[FACT] Certain AWS costs **cannot be allocated by tag**:
- RI/SP commitment fees (subscription fees) have no tag at purchase time
- Data Transfer fees do not always have a tag
- Support Plan is billed at the account level (not per resource)

[FACT] For RI/SP, it is possible to track *how discounts were applied* to each resource/tag in Cost Explorer (using AmortizedCost by tag), but the **purchase fee** itself is not taggable.

---

## 10. Diagram: Tag Governance Pipeline

```
                TAG COMPLIANCE PIPELINE

  Resource Creation            Detection                   Remediation
  ─────────────────            ─────────                   ──────────
  IaC (CDK/Terraform)
    → tags in template    AWS Config Rule               Tag Editor API
                          required-tags         ──────► tag-resources in batch
  Console/API direct    ──► evaluates on
    → SCP blocks if           each change
      without "project"       ↓
      tag (Deny + Null)   NON_COMPLIANT            Notification to Owner
                          → SNS → Lambda   ──────► email "resource without tag"
                              ↓                    slack #finops-alerts
  Tag Policy (Org)        EventBridge Rule
    → enforce lowercase   aws.config →
    → enforce allowed      Config change
      values                (via SNS)
                              ↓
                          CloudWatch            Weekly Report
                          Dashboard      ──────► % compliant resources
                                                 top 10 non-compliant
                                                 unallocated cost $
```

---

## 11. Pitfalls

[FACT] **Budget Actions do not automatically revert**: when applying an SCP or IAM policy, the action remains active even after costs drop. It must be manually reverted via `execute-budget-action` with `REVERSE_BUDGET_ACTION`.

[FACT] **SCPs can only be applied by the management account**: if the budget is in a member account, the `APPLY_SCP_POLICY` Budget Action type will not work. Only the management account can create/attach SCPs.

[FACT] **Tag Policies do not block, only report**: Tag Policies (Organizations) generate a compliance report but **do not prevent** creation of resources with incorrect values. To block, combine with an SCP.

[FACT] **OU/Account tags in Organizations do not work in Cost Explorer**: tags applied to OUs or Organization accounts are not usable for filtering in Cost Explorer. Use Cost Categories to aggregate accounts into cost groups.

[CONSENSUS] **High tag cardinality increases billing cost**: each unique combination of active tags generates a line in the Cost and Usage Report (CUR). Tags like `Version` (changes every deploy) or `InstanceId` (too specific) can cause an explosion of lines in the CUR. Reserve Cost Allocation Tags for stable dimensions (project, team, environment).

[FACT] **Budget FORECASTED may not trigger**: the forecast is only calculated after sufficient data from the current month. At the beginning of the month, the model has little historical basis and may underestimate the forecast.

[UNCERTAIN] **Limit of 500 activated Cost Allocation Tags**: there is a documented limit of 500 activated tags per management account in the Organization (verify in your organization — may have been updated).

---

## Reflection Exercise

A company has 200 AWS accounts in an Organization with a total spend of $500K/month. The CFO wants to implement chargeback for 8 different business units, but there are three problems: (1) only 60% of resources have `project` and `team` tags; (2) RI and Savings Plans were purchased in the management account without BU separation; (3) shared infrastructure resources (VPC, Route53, IAM) cannot be tagged by BU.

**Propose a complete cost allocation strategy**, answering:

1. Which of the three models (account-based, team-based, tag-based) — or combination — would you adopt? Why?
2. How to handle the 40% of resources without tags? What sequence of actions for remediation without disrupting production?
3. How to allocate the cost of RI/SP that was purchased centrally among the 8 BUs?
4. How to treat shared infrastructure costs (VPC, DNS, NAT Gateway) — what does the whitepaper suggest?
5. How to use Budget Actions to create automatic guardrails **without risk** of shutting down production?

---

## References

- [FACT] [Configuring budget actions](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-controls.html) — docs.aws.amazon.com
- [FACT] [Configuring a budget action](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-action-configure.html) — docs.aws.amazon.com
- [FACT] [Building a cost allocation strategy](https://docs.aws.amazon.com/whitepapers/latest/tagging-best-practices/building-a-cost-allocation-strategy.html) — docs.aws.amazon.com (whitepaper)
- [FACT] [Implementing and enforcing tagging](https://docs.aws.amazon.com/whitepapers/latest/tagging-best-practices/implementing-and-enforcing-tagging.html) — docs.aws.amazon.com (whitepaper)
- [FACT] [Cost allocation tags](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/cost-alloc-tags.html) — docs.aws.amazon.com
- [FACT] [Tags for cost allocation and financial management](https://docs.aws.amazon.com/whitepapers/latest/tagging-best-practices/tags-for-cost-allocation-and-financial-management.html) — docs.aws.amazon.com (whitepaper)
