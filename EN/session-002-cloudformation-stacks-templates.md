# Session 002 — CloudFormation: stacks, templates, parameters, outputs, Ref/GetAtt

**Estimated duration:** 60 minutes
**Prerequisites:** session-001 — Advanced AWS CLI (SSO, profiles, assume-role)

---

## Objective

By the end, you will be able to write a complete YAML template with typed Parameters, Resources referencing others with `Ref` and `Fn::GetAtt`, Outputs exported between stacks, and deploy via `aws cloudformation deploy` with changesets.

---

## Context

[FACT] CloudFormation is AWS's native IaC service since 2011. It operates on the declarative model: you describe the desired state of infrastructure in a template, and CloudFormation calculates the delta between the current stack state and the desired one, executing the necessary operations in the correct order.

[CONSENSUS] Even if you use CDK (which comes in the next sessions), understanding raw CloudFormation is essential — CDK generates CloudFormation as its output artifact, and any deploy debugging happens at the CloudFormation level. Engineers who skip this layer are blind when something goes wrong in `cdk deploy`.

The central distinction of this session is: a template is the document, a stack is the running instance. The same template can create multiple stacks in different regions or accounts.

---

## Core concepts

### 1. Template anatomy

[FACT] A CloudFormation template has 10 possible sections. Only `Resources` is mandatory.

```yaml
AWSTemplateFormatVersion: "2010-09-09"   # fixed, never changes
Description: "What this stack does"

Metadata: {}        # data for external tools (e.g., Console UI hints)
Parameters: {}      # user inputs when creating/updating the stack
Rules: {}           # parameter validations (cross-parameter constraints)
Mappings: {}        # static lookup tables (e.g., AMI by region)
Conditions: {}      # boolean flags based on parameters
Transform: {}       # macros (e.g., SAM uses AWS::Serverless-2016-10-31)
Resources: {}       # REQUIRED — resources to create
Outputs: {}         # values to export or display
```

**CloudFormation processing order:**

```
1. Parameters   → resolves inputs
2. Rules        → validates parameter combinations
3. Mappings     → makes lookup tables available
4. Conditions   → evaluates boolean flags
5. Resources    → determines creation order via dependency graph
6. Outputs      → resolves references after resource creation
```

This order matters because `Ref` and `Fn::GetAtt` in `Outputs` only work after resources exist.

---

### 2. Parameters — typing, constraints, and best practices

Parameters receive values at runtime. Unlike environment variables, they are validated by CloudFormation before any resource is created.

**Available types:**

```yaml
Parameters:

  # Primitive types
  Env:
    Type: String
    AllowedValues: [dev, staging, prod]
    Default: dev

  InstanceType:
    Type: String
    AllowedPattern: "^(t3|m5|c5)\\.(micro|small|medium|large)$"
    ConstraintDescription: "Only t3, m5 or c5 instances"

  Port:
    Type: Number
    MinValue: 1024
    MaxValue: 65535

  # AWS types — CloudFormation validates the resource's existence in the account
  VpcId:
    Type: AWS::EC2::VPC::Id           # dropdown in console, validated

  SubnetIds:
    Type: List<AWS::EC2::Subnet::Id>  # list of subnets

  AmiId:
    Type: AWS::EC2::Image::Id

  # SSM Parameter Store — resolves the value at runtime
  LatestAmiId:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64
```

[CONSENSUS] Using `AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>` for AMIs is the recommended practice — the template always uses the most recent AMI without needing to be updated. The downside is that the deploy can change without any template modification, which complicates drift tracking.

**Referencing parameters:**

```yaml
Resources:
  MyInstance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref InstanceType   # !Ref returns the parameter value
      ImageId: !Ref LatestAmiId
```

---

### 3. Resources — the heart of the template

Each resource has: logical ID (name in the template), Type, and Properties.

```yaml
Resources:

  # Logical ID: internal name — referenced by Ref/GetAtt
  # Convention: descriptive PascalCase (AppBucket, WebServerSG, etc.)
  AppBucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain             # Snapshot | Retain | Delete (default)
    UpdateReplacePolicy: Retain        # behavior when replacing the resource
    DependsOn: LogGroup                # forces explicit order (use only when necessary)
    Properties:
      BucketName: !Sub "app-${Env}-${AWS::AccountId}"
      VersioningConfiguration:
        Status: Enabled
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256

  WebServerSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: "Web server security group"
      VpcId: !Ref VpcId
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0

  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref InstanceType
      ImageId: !Ref LatestAmiId
      SecurityGroupIds:
        - !Ref WebServerSG          # Ref on a resource returns the physical ID
      Tags:
        - Key: Name
          Value: !Sub "web-${Env}"
```

**`DeletionPolicy` and `UpdateReplacePolicy` — critical difference:**

```
DeletionPolicy       → what happens when the STACK is deleted
UpdateReplacePolicy  → what happens when the resource needs to be REPLACED
                       (e.g., changing the name of an S3 bucket forces replacement)

Values:
  Delete    → resource is deleted (default for most)
  Retain    → resource remains in the account, disassociated from the stack
  Snapshot  → takes a snapshot before deleting (RDS, EBS, ElastiCache)
```

[FACT] `DeletionPolicy: Retain` on an S3 bucket does not prevent CloudFormation from attempting to delete the bucket when deleting the stack — it only causes CloudFormation to "forget" the resource instead of deleting it. If the bucket has objects and does not have `Retain`, the stack delete will fail with `BucketNotEmpty`.

---

### 4. Ref and Fn::GetAtt — what each one returns

This is the biggest source of confusion in CloudFormation. `Ref` and `Fn::GetAtt` return different things depending on the resource type.

**`Ref` — returns the resource's primary identifier:**

```yaml
# Each resource type has a different "Ref value":
!Ref AppBucket        # → bucket name (e.g., "app-prod-123456789")
!Ref WebServerSG      # → Group ID (e.g., "sg-0abc123")
!Ref WebServer        # → Instance ID (e.g., "i-0abc123")
!Ref MyQueue          # → Queue URL (e.g., "https://sqs.us-east-1...")
!Ref MyTable          # → DynamoDB table name
!Ref MyFunction       # → Lambda function name
!Ref MyRole           # → IAM role name (NOT the ARN)
```

[FACT] To know what `Ref` returns for a specific type, consult the resource documentation under "Return values > Ref". There is no universal rule — each service defines what its "primary identifier" is.

**`Fn::GetAtt` — returns specific attributes:**

```yaml
# Long syntax
Fn::GetAtt:
  - AppBucket
  - Arn

# Short syntax (YAML)
!GetAtt AppBucket.Arn           # → arn:aws:s3:::app-prod-123456789
!GetAtt AppBucket.DomainName    # → app-prod-123456789.s3.amazonaws.com
!GetAtt WebServerSG.GroupId     # → sg-0abc123 (same as Ref here)
!GetAtt WebServer.PrivateIp     # → 10.0.1.50
!GetAtt WebServer.PublicIp      # → 3.x.x.x (if it has a public IP)
!GetAtt MyRole.Arn              # → arn:aws:iam::123:role/MyRole
!GetAtt MyFunction.Arn          # → arn:aws:lambda:us-east-1:123:function:name
```

**Diagram of the difference:**

```
Resource: AWS::S3::Bucket (logical ID: AppBucket)
         │
         ├── Ref           → bucket name     "app-prod-123456789012"
         ├── GetAtt .Arn   → full ARN        "arn:aws:s3:::app-prod-..."
         └── GetAtt .DomainName → endpoint   "app-prod-....s3.amazonaws.com"

Resource: AWS::IAM::Role (logical ID: MyRole)
         │
         ├── Ref           → role name       "MyRole-XXXXXXXXXXX"
         └── GetAtt .Arn   → full ARN        "arn:aws:iam::123:role/MyRole-..."
         (for IAM Roles you almost always want the ARN → use GetAtt, not Ref)
```

**Most commonly used string functions with Ref/GetAtt:**

```yaml
# Sub — string interpolation (more readable than Join)
!Sub "arn:aws:s3:::${AppBucket}/*"
!Sub "https://${WebServer.PublicIp}:443"
!Sub "${AWS::StackName}-${AWS::Region}-logs"   # pseudo-parameters

# Join — joins list with delimiter
!Join [",", [!Ref SubnetA, !Ref SubnetB]]

# Select — gets list element by index
!Select [0, !GetAZs ""]   # first AZ of the current region

# Pseudo-parameters always available (no need to declare in Parameters):
# AWS::AccountId, AWS::Region, AWS::StackName, AWS::StackId, AWS::NoValue
```

---

### 5. Outputs and cross-stack references

Outputs serve two distinct purposes: displaying useful values after deploy, and exporting values for other stacks to consume.

**Output structure:**

```yaml
Outputs:

  BucketName:
    Description: "Application bucket name"
    Value: !Ref AppBucket          # value displayed in console and returned by API

  BucketArn:
    Description: "Bucket ARN"
    Value: !GetAtt AppBucket.Arn
    Export:
      Name: !Sub "${AWS::StackName}-BucketArn"   # unique name in the region/account
```

**Consuming an export in another stack:**

```yaml
# Stack B imports the value exported by Stack A
Resources:
  LambdaFunction:
    Type: AWS::Lambda::Function
    Properties:
      Environment:
        Variables:
          BUCKET_ARN: !ImportValue "my-infra-stack-BucketArn"
```

**Critical Export/ImportValue restrictions — [FACT]:**

```
1. Export names are unique per region/account — there are no namespaces
2. You CANNOT delete a stack that has exported outputs
   while another stack is using those exports via ImportValue
3. You CANNOT modify or remove an exported output
   while it is being imported
4. Exports do NOT work cross-region — only cross-stack in the same region
5. The Export Name CANNOT use Ref/GetAtt from resources
   (can use !Sub with pseudo-parameters like ${AWS::StackName})
```

[OPINION — Alex Pulver, AWS CDK team] For complex architectures, the coupling created by Export/ImportValue makes operations like delete and refactoring difficult. An alternative is using SSM Parameter Store to share values between stacks, maintaining decoupling.

---

### 6. `aws cloudformation deploy` — what happens under the hood

`aws cloudformation deploy` is a high-level wrapper that:
1. Creates a changeset (never applies directly)
2. Waits for the changeset to be computed
3. Executes the changeset automatically
4. Waits for the stack to reach `CREATE_COMPLETE` or `UPDATE_COMPLETE`

```bash
# Basic deploy
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name my-stack \
  --parameter-overrides Env=prod InstanceType=t3.medium \
  --capabilities CAPABILITY_IAM \
  --profile prod

# View the changeset before executing (does not apply)
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name my-stack \
  --no-execute-changeset

# View the generated changeset
aws cloudformation describe-change-set \
  --stack-name my-stack \
  --change-set-name <changeset-name>

# Execute manually after review
aws cloudformation execute-change-set \
  --stack-name my-stack \
  --change-set-name <changeset-name>
```

**`--capabilities` — when to use:**

```
CAPABILITY_IAM          → template creates IAM roles or policies without custom name
CAPABILITY_NAMED_IAM    → template creates roles with explicit name (more permissive)
CAPABILITY_AUTO_EXPAND  → template uses Transform (SAM, custom macros)
```

[FACT] Without the correct capability, the deploy fails with `InsufficientCapabilitiesException`. This is intentional — it's a protection against accidental creation of IAM resources.

**`aws cloudformation deploy` flow diagram:**

```
aws cloudformation deploy
        │
        ▼
  Stack exists?
    │         │
   No         Yes
    │          │
    ▼          ▼
CREATE_     UPDATE_
CHANGESET   CHANGESET
    │          │
    └────┬─────┘
         ▼
   Waits for REVIEW_IN_PROGRESS
         │
         ▼
   Executes changeset
         │
         ▼
   Waits for CREATE_COMPLETE
      or UPDATE_COMPLETE
         │
   Error? → ROLLBACK_COMPLETE
           (stack returns to previous state)
```

**Most important stack statuses:**

```
CREATE_IN_PROGRESS    → creation in progress
CREATE_COMPLETE       → creation completed successfully
CREATE_FAILED         → creation failed (stack remains)
ROLLBACK_IN_PROGRESS  → rollback in progress after failure
ROLLBACK_COMPLETE     → rollback completed (stack exists but empty)
UPDATE_IN_PROGRESS    → update in progress
UPDATE_COMPLETE       → update completed
UPDATE_ROLLBACK_*     → update rollback
DELETE_IN_PROGRESS    → deletion in progress
DELETE_FAILED         → deletion failed (resource could not be deleted)
```

---

## Practical example

Complete template for an application with S3 bucket + IAM role + value export:

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: "Base infrastructure — app bucket + access role"

Parameters:
  Env:
    Type: String
    AllowedValues: [dev, staging, prod]
    Default: dev
  
  ProjectName:
    Type: String
    MinLength: 3
    MaxLength: 20
    AllowedPattern: "^[a-z][a-z0-9-]+$"
    ConstraintDescription: "Lowercase, starts with letter, only letters/numbers/hyphens"

Resources:
  AppBucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: !If [IsProd, Retain, Delete]
    Properties:
      BucketName: !Sub "${ProjectName}-${Env}-${AWS::AccountId}-${AWS::Region}"
      VersioningConfiguration:
        Status: !If [IsProd, Enabled, Suspended]
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true

  AppBucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref AppBucket
      PolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Sid: DenyNonTLS
            Effect: Deny
            Principal: "*"
            Action: s3:*
            Resource:
              - !GetAtt AppBucket.Arn
              - !Sub "${AppBucket.Arn}/*"
            Condition:
              Bool:
                aws:SecureTransport: false

  AppRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub "${ProjectName}-${Env}-app-role"
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
      Policies:
        - PolicyName: BucketAccess
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: Allow
                Action:
                  - s3:GetObject
                  - s3:PutObject
                  - s3:DeleteObject
                Resource: !Sub "${AppBucket.Arn}/*"

Conditions:
  IsProd: !Equals [!Ref Env, prod]

Outputs:
  BucketName:
    Description: "Bucket name"
    Value: !Ref AppBucket
    Export:
      Name: !Sub "${AWS::StackName}-BucketName"

  BucketArn:
    Description: "Bucket ARN"
    Value: !GetAtt AppBucket.Arn
    Export:
      Name: !Sub "${AWS::StackName}-BucketArn"

  AppRoleArn:
    Description: "Application role ARN"
    Value: !GetAtt AppRole.Arn
    Export:
      Name: !Sub "${AWS::StackName}-AppRoleArn"
```

**Deploy:**

```bash
# Validate the template before submitting
aws cloudformation validate-template --template-body file://template.yaml

# Deploy to dev
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name my-project-infra \
  --parameter-overrides Env=dev ProjectName=my-project \
  --capabilities CAPABILITY_NAMED_IAM \
  --profile dev

# View the outputs after deploy
aws cloudformation describe-stacks \
  --stack-name my-project-infra \
  --query 'Stacks[0].Outputs' \
  --output table

# View all available exports in the account/region
aws cloudformation list-exports --query 'Exports[*].[Name,Value]' --output table
```

---

## Common pitfalls

**1. Using `Ref` on IAM Roles when you need the ARN**

`!Ref MyRole` returns the role name, not the ARN. When you pass this to a field that expects an ARN (like `RoleArn` of an ECS TaskDefinition), CloudFormation accepts the template but the service returns an error at runtime. Always use `!GetAtt MyRole.Arn` for ARNs.

**2. Deleting a stack with exports in use**

If another stack uses `!ImportValue "my-stack-BucketArn"`, you cannot delete `my-stack` nor modify that output. CloudFormation returns `Export cannot be updated as it is in use by another stack`. To resolve: first remove the `ImportValue` from the consuming stack, deploy it, then delete or modify the exporting stack.

**3. Unnecessary `DependsOn`**

When you use `!Ref` or `!GetAtt` from one resource in another, CloudFormation already infers the dependency automatically. Explicit `DependsOn` is only necessary when a resource depends on another without directly referencing it — for example, a Lambda function that queries an endpoint that only exists after an EC2 instance is running.

---

## Reflection exercise

You are migrating an existing architecture to CloudFormation and need to split the infrastructure into two stacks: `infra-base` (VPC, subnets, security groups) and `infra-app` (ECS service, ALB, IAM roles).

The `infra-app` stack needs to reference the VPC ID and subnet IDs created by `infra-base`. A colleague proposes using `Export/ImportValue`. Another proposes using SSM Parameter Store as an intermediary (the `infra-base` writes values to SSM, the `infra-app` reads them via `AWS::SSM::Parameter::Value`).

What are the real trade-offs of each approach? Consider: stack lifecycle, ease of debugging, future refactoring operations, and what happens when you need to recreate `infra-base` from scratch in a new account. Which would you choose and why?

---

## Resources for deeper learning

**Template anatomy and sections:**
- [CloudFormation template sections](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-anatomy.html) — complete reference of all sections with examples of each field.

**Intrinsic functions:**
- [Intrinsic function reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/intrinsic-function-reference.html) — lists all functions with what they return per resource type. Consult here whenever you have doubt about what `Ref` returns for a specific resource.

**Cross-stack references:**
- [Refer to resource outputs in another CloudFormation stack](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/walkthrough-crossstackref.html) — complete walkthrough of Export/ImportValue with documented failure scenarios.

**cfn-lint:**
- [cfn-lint on GitHub](https://github.com/aws-cloudformation/cfn-lint) — static linter for CloudFormation templates. Detects wrong types, invalid references and nonexistent properties before deploy. Install with `pip install cfn-lint` and run with `cfn-lint template.yaml`.

---
