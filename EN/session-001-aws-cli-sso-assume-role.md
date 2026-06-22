# Session 001 — Advanced AWS CLI: SSO, profiles, assume-role, and pagination

**Estimated duration:** 60 minutes
**Prerequisites:** none

---

## Objective

By the end, you will be able to configure named profiles with SSO via `aws configure sso`, perform `aws sts assume-role` for context switching between accounts, paginate results with `--page-size` and `--max-items` without losing records, and use `--query` with JMESPath to filter outputs.

---

## Context

[FACT] AWS CLI v2 introduced native support for IAM Identity Center (formerly AWS SSO) with automatic token management, eliminating the need for external tools like `aws-vault` for the basic authentication flow. Before this, the only way to use temporary credentials with the CLI was to manually export the environment variables returned by `aws sts assume-role`.

[CONSENSUS] For multi-account environments — which is the standard in organizations with mature AWS usage — the recommended authentication flow today is SSO via IAM Identity Center, with named profiles in `~/.aws/config` for each account+role combination. Manual `aws sts assume-role` still has legitimate use in automations and pipelines, but has lost ground as the primary flow for engineers.

This guide covers the three pillars every AWS engineer uses daily: authenticating via SSO, switching context between accounts, and extracting data from APIs without losing records or overloading the API.

---

## Core concepts

### 1. AWS CLI credential architecture

The CLI resolves credentials in a well-defined priority chain ([FACT], per official documentation):

```
1. Environment variables        AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_SESSION_TOKEN
2. Default profile in ~/.aws    [default] in ~/.aws/credentials or ~/.aws/config
3. Container credentials        (ECS task role via ECS_CONTAINER_METADATA_URI)
4. Instance profile             (EC2 instance role via IMDS)
```

The `~/.aws/config` file is the correct place for named profiles — including SSO. The `~/.aws/credentials` file is legacy and only carries `aws_access_key_id` and `aws_secret_access_key`. [CONSENSUS] In modern environments, ideally `~/.aws/credentials` should be empty or nonexistent.

```
~/.aws/
├── config          # profiles, SSO, roles — everything goes here
└── credentials     # legacy; avoid using for new profiles
```

The distinction between the two files matters because the CLI's `--profile` flag reads both, but `config` supports richer features (SSO, source_profile, role chaining, MFA serial).

---

### 2. SSO with IAM Identity Center — the `sso-session` model

[FACT] AWS CLI v2.x introduced the concept of `sso-session`, which is a separate section in `~/.aws/config` responsible for managing the SSO access token centrally. Before this, each profile needed to have the SSO configurations repeated — and the token was not shared.

**Current structure (v2, recommended):**

```ini
# ~/.aws/config

[sso-session my-org]
sso_start_url    = https://my-org.awsapps.com/start
sso_region       = us-east-1
sso_registration_scopes = sso:account:access

[profile dev]
sso_session      = my-org
sso_account_id   = 111122223333
sso_role_name    = DeveloperAccess
region           = us-east-1
output           = json

[profile prod-readonly]
sso_session      = my-org
sso_account_id   = 444455556666
sso_role_name    = ReadOnlyAccess
region           = us-east-1
output           = json
```

The `[sso-session]` block is shared across all profiles from the same org. A single `aws sso login --sso-session my-org` authenticates all profiles simultaneously.

**Authentication flow:**

```
aws configure sso
  └── creates the sso-session and the profile in ~/.aws/config

aws sso login --profile dev
  └── opens the browser → authorizes in Identity Center
  └── saves the token in ~/.aws/sso/cache/<hash>.json

aws s3 ls --profile dev
  └── CLI reads the token from cache
  └── exchanges it for temporary credentials via STS
  └── credentials are stored in ~/.aws/cli/cache/<hash>.json (TTL = role duration)
```

**Token cache and automatic refresh:**

[FACT] The SSO token is stored in `~/.aws/sso/cache/`. The CLI v2 checks the token every hour and uses the refresh token to renew it automatically within the extended session period configured in Identity Center (default: 8h, configurable maximum: 90 days). When the token expires completely, the command fails with `Token has expired and refresh failed` — in that case, `aws sso login` resolves it.

```bash
# View the cached token (useful for debugging)
cat ~/.aws/sso/cache/*.json | jq .

# Revoke and re-authenticate
aws sso logout
aws sso login --profile dev
```

**Complete SSO flow diagram:**

```
[You]
  │
  ├─ aws sso login ──► [Browser: Identity Center]
  │                         │
  │                    Authorizes access
  │                         │
  │◄──────── access_token + refresh_token ──────────
  │          (in ~/.aws/sso/cache/)
  │
  ├─ aws ec2 describe-instances --profile dev
  │         │
  │    CLI reads access_token from SSO cache
  │         │
  │    ┌────▼────────────────────┐
  │    │ STS: AssumeRoleWithWebIdentity │
  │    └────┬────────────────────┘
  │         │
  │    temporary credentials (15min–12h)
  │    (in ~/.aws/cli/cache/)
  │         │
  │    ┌────▼────────────────┐
  │    │ EC2 API call        │
  │    └─────────────────────┘
```

---

### 3. `aws sts assume-role` — context switching between accounts

`assume-role` is the STS operation that exchanges current credentials for temporary credentials of another role. It is the mechanism behind all cross-account access.

**Direct usage (imperative — good for scripts):**

```bash
# Assume the role and extract credentials
CREDS=$(aws sts assume-role \
  --role-arn arn:aws:iam::999988887777:role/DeployRole \
  --role-session-name my-session \
  --duration-seconds 3600 \
  --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]' \
  --output text)

export AWS_ACCESS_KEY_ID=$(echo $CREDS | awk '{print $1}')
export AWS_SECRET_ACCESS_KEY=$(echo $CREDS | awk '{print $2}')
export AWS_SESSION_TOKEN=$(echo $CREDS | awk '{print $3}')

# Verify that you are operating as the assumed role
aws sts get-caller-identity
```

**Declarative usage via profile (recommended for interactive use):**

```ini
# ~/.aws/config
[profile deploy-prod]
role_arn       = arn:aws:iam::999988887777:role/DeployRole
source_profile = dev          # uses credentials from the 'dev' profile to assume
role_session_name = luiz-cli
duration_seconds = 3600
region         = us-east-1
```

```bash
aws sts get-caller-identity --profile deploy-prod
# The CLI performs the AssumeRole automatically, without any script
```

**Role chaining — and its critical limit:**

[FACT] Role chaining is when you use the credentials of an assumed role to assume another role. The duration limit is **fixed at 1 hour**, regardless of the value passed in `--duration-seconds`. This is an STS restriction, not a CLI one.

```
IAM User → Assumes RoleA (duration: up to 12h)    ✅ OK
RoleA    → Assumes RoleB (duration: max 1h)       ⚠️ Always 1h
RoleB    → Assumes RoleC (duration: max 1h)       ⚠️ Always 1h
```

If you try `--duration-seconds 7200` in a chain, STS returns an error. The design is intentional — AWS limits the "depth" of delegation to reduce attack surface.

```ini
# Example of chain: dev → staging → prod
[profile staging]
role_arn       = arn:aws:iam::222233334444:role/StagingAccess
source_profile = dev

[profile prod]
role_arn       = arn:aws:iam::555566667777:role/ProdAccess
source_profile = staging      # chain: uses staging to reach prod
```

**MFA in an assume-role:**

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::999988887777:role/PrivilegedRole \
  --role-session-name mfa-session \
  --serial-number arn:aws:iam::111122223333:mfa/luiz \
  --token-code 123456
```

```ini
# Or declarative in config:
[profile privileged]
role_arn       = arn:aws:iam::999988887777:role/PrivilegedRole
source_profile = default
mfa_serial     = arn:aws:iam::111122223333:mfa/luiz
```

---

### 4. Pagination — `--page-size` vs `--max-items`

This is one of the most common confusions in the CLI. The two parameters control different things.

```
--page-size   = size of each request to the AWS API
--max-items   = total number of items in the CLI output
```

**Flow diagram:**

```
AWS API has 500 S3 objects

Without pagination:
  CLI makes 1 request → returns 500 items (may timeout)

With --page-size 50:
  CLI makes 10 requests of 50 items each → output: 500 items
  (protects against timeout, transparent to the user)

With --max-items 100:
  CLI makes requests until it has 100 items → stops
  output: 100 items + NextToken to continue

With --page-size 50 --max-items 100:
  CLI makes 2 requests of 50 items → stops
  output: 100 items + NextToken
```

**The silent bug:**

[FACT] If `--max-items` is not a multiple of `--page-size`, you can get duplicate items or lose items. Classic example: `--page-size 30 --max-items 50` — the CLI fetches 2 pages of 30 (60 items), returns the first 50, but the `NextToken` points to item 31 of the second page, not item 51 of the total. When using `--starting-token`, you restart from item 31, losing items 51–60.

**Safe recommendation:**

```bash
# 1. To list everything without risk: use only --page-size to control throughput
aws s3api list-objects-v2 \
  --bucket my-bucket \
  --page-size 100

# 2. To paginate manually with explicit cursor
aws s3api list-objects-v2 \
  --bucket my-bucket \
  --page-size 50 \
  --max-items 50

# Get the NextToken from the previous output and continue:
aws s3api list-objects-v2 \
  --bucket my-bucket \
  --page-size 50 \
  --max-items 50 \
  --starting-token "eyJ..."

# 3. To avoid timeout on huge buckets (smaller page = smaller requests)
aws s3api list-objects-v2 \
  --bucket bucket-with-millions-of-objects \
  --page-size 20
```

---

### 5. `--query` with JMESPath

`--query` applies a JMESPath expression to the JSON response before output. It is processed locally (no additional API cost).

**Essential syntax:**

```bash
# Simple field access
aws ec2 describe-instances \
  --query 'Reservations[0].Instances[0].InstanceId'

# Projection: get specific fields from a list
aws ec2 describe-instances \
  --query 'Reservations[].Instances[].[InstanceId,State.Name,Tags[?Key==`Name`].Value|[0]]' \
  --output table

# Filter: only running instances
aws ec2 describe-instances \
  --query 'Reservations[].Instances[?State.Name==`running`].[InstanceId,PrivateIpAddress]' \
  --output text

# Pipe: get only the first result from a filtered list
aws ec2 describe-instances \
  --query 'Reservations[].Instances[?State.Name==`running`].InstanceId | [0]'

# Flatten nested lists with []
aws ec2 describe-instances \
  --query 'Reservations[].Instances[].InstanceId'
  # Without the inner [] you would get [[id1, id2], [id3]] instead of [id1, id2, id3]
```

**Combining pagination with query:**

```bash
# List all Lambda functions with Python runtime
aws lambda list-functions \
  --page-size 50 \
  --query 'Functions[?Runtime==`python3.12`].[FunctionName,CodeSize]' \
  --output table
```

---

## Practical example

**Scenario:** you have SSO access to the development account and need to list all EC2 instances in production (another account), filtering only those that are `running`, without downloading the entire list at once.

```bash
# 1. Authenticate via SSO (once per session)
aws sso login --profile dev

# 2. Verify current identity
aws sts get-caller-identity --profile dev

# 3. Assume the prod role via chained profile
# (already configured in ~/.aws/config with source_profile = dev)
aws sts get-caller-identity --profile prod-readonly

# 4. List running instances in prod, paginating 20 at a time
aws ec2 describe-instances \
  --profile prod-readonly \
  --region us-east-1 \
  --page-size 20 \
  --query 'Reservations[].Instances[?State.Name==`running`].[InstanceId,InstanceType,Tags[?Key==`Name`].Value|[0]]' \
  --output table

# 5. If the environment has many instances and you want to paginate manually
RESULT=$(aws ec2 describe-instances \
  --profile prod-readonly \
  --page-size 20 \
  --max-items 20 \
  --output json)

echo $RESULT | jq '.NextToken'   # token for the next page

aws ec2 describe-instances \
  --profile prod-readonly \
  --page-size 20 \
  --max-items 20 \
  --starting-token "$(echo $RESULT | jq -r '.NextToken')" \
  --output json
```

---

## Common pitfalls

**1. `--max-items` without aligned `--page-size` — silently missing data**

The mistake is using `--max-items 100` with `--page-size 30` and then trying to continue with `--starting-token`. The cursor gets desynchronized with page boundaries and you lose items between pages. Solution: use the same value for both, or use only `--page-size` to list everything.

**2. Confusing `~/.aws/credentials` with `~/.aws/config` for SSO profiles**

SSO profiles **do not work** in `~/.aws/credentials`. The `credentials` file only understands `aws_access_key_id` and `aws_secret_access_key`. If you put `sso_session` there, the CLI silently ignores it and falls through to the next provider in the chain.

**3. Role chaining with `duration_seconds > 3600`**

If a profile uses `source_profile` pointing to another profile (not to direct IAM user credentials), you are doing role chaining. Any `duration_seconds` above 3600 will fail with `DurationSeconds exceeds the 1 hour session duration limit for roles assumed by role chaining`. The error is not obvious if you don't know you're in a chain.

---

## Reflection exercise

You have three AWS accounts: `tools` (where IAM Identity Center is), `staging` and `prod`. An engineer configures the following `~/.aws/config`:

```ini
[sso-session company]
sso_start_url = https://company.awsapps.com/start
sso_region    = us-east-1

[profile staging]
sso_session    = company
sso_account_id = 111122223333
sso_role_name  = DevAccess
region         = us-east-1

[profile prod]
role_arn       = arn:aws:iam::444455556666:role/ProdDeploy
source_profile = staging
duration_seconds = 7200
region         = us-east-1
```

When they run `aws sts get-caller-identity --profile prod`, the command fails. Why? What are the two distinct problems in this configuration — one architectural, one a service limit — and how would you fix each one? Also consider the impact on the session duration in prod if the configuration is corrected.

---

## Resources for deeper learning

**Configuration and profiles:**
- [Configuring IAM Identity Center authentication with the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sso.html) — primary reference for the `sso-session` model, includes all accepted fields and examples of interactive `aws configure sso`.

**Assume-role and role chaining:**
- [Using an IAM role in the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-role.html) — covers `source_profile`, `credential_source`, MFA and role chaining limits with config examples.

**Pagination:**
- [Using the pagination options in the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-usage-pagination.html) — short and direct document with the distinction between `--page-size` and `--max-items` and how to use `--starting-token`.

---
