# Sessão 002 — CloudFormation: stacks, templates, parameters, outputs, Ref/GetAtt

**Duração estimada:** 60 minutos
**Pré-requisitos:** session-001 — AWS CLI avançado (SSO, perfis, assume-role)

---

## Objetivo

Ao final, você conseguirá escrever um template YAML completo com Parameters tipados, Resources referenciando outros com `Ref` e `Fn::GetAtt`, Outputs exportados entre stacks, e fazer deploy via `aws cloudformation deploy` com changesets.

---

## Contexto

[FATO] O CloudFormation é o serviço nativo de IaC da AWS desde 2011. Ele opera no modelo declarativo: você descreve o estado desejado da infraestrutura num template, e o CloudFormation calcula o delta entre o estado atual da stack e o desejado, executando as operações necessárias na ordem correta.

[CONSENSO] Mesmo que você use CDK (que vem nas próximas sessões), entender CloudFormation puro é essencial — o CDK gera CloudFormation como artefato de saída, e qualquer debugging de deploy acontece no nível do CloudFormation. Engenheiros que pulam essa camada ficam cegos quando algo dá errado no `cdk deploy`.

A distinção central desta sessão é: template é o documento, stack é a instância em execução. O mesmo template pode criar múltiplas stacks em regiões ou contas diferentes.

---

## Conceitos principais

### 1. Anatomia do template

[FATO] Um template CloudFormation tem 10 seções possíveis. Apenas `Resources` é obrigatória.

```yaml
AWSTemplateFormatVersion: "2010-09-09"   # fixo, nunca muda
Description: "O que essa stack faz"

Metadata: {}        # dados para ferramentas externas (ex: Console UI hints)
Parameters: {}      # inputs do usuário ao criar/atualizar a stack
Rules: {}           # validações de parâmetros (cross-parameter constraints)
Mappings: {}        # lookup tables estáticas (ex: AMI por região)
Conditions: {}      # flags booleanas baseadas em parâmetros
Transform: {}       # macros (ex: SAM usa AWS::Serverless-2016-10-31)
Resources: {}       # OBRIGATÓRIO — recursos a criar
Outputs: {}         # valores a exportar ou exibir
```

**Ordem de processamento do CloudFormation:**

```
1. Parameters   → resolve inputs
2. Rules        → valida combinações de parâmetros
3. Mappings     → disponibiliza lookup tables
4. Conditions   → avalia flags booleanas
5. Resources    → determina ordem de criação pelo grafo de dependências
6. Outputs      → resolve referências após criação dos recursos
```

Essa ordem importa porque `Ref` e `Fn::GetAtt` em `Outputs` só funcionam depois que os recursos existem.

---

### 2. Parameters — tipagem, constraints e boas práticas

Parameters recebem valores em runtime. Ao contrário de variáveis de ambiente, eles são validados pelo CloudFormation antes de qualquer recurso ser criado.

**Tipos disponíveis:**

```yaml
Parameters:

  # Tipos primitivos
  Env:
    Type: String
    AllowedValues: [dev, staging, prod]
    Default: dev

  InstanceType:
    Type: String
    AllowedPattern: "^(t3|m5|c5)\\.(micro|small|medium|large)$"
    ConstraintDescription: "Apenas instâncias t3, m5 ou c5"

  Port:
    Type: Number
    MinValue: 1024
    MaxValue: 65535

  # Tipos AWS — CloudFormation valida a existência do recurso na conta
  VpcId:
    Type: AWS::EC2::VPC::Id           # dropdown no console, validado

  SubnetIds:
    Type: List<AWS::EC2::Subnet::Id>  # lista de subnets

  AmiId:
    Type: AWS::EC2::Image::Id

  # SSM Parameter Store — resolve o valor em runtime
  LatestAmiId:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64
```

[CONSENSO] Usar `AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>` para AMIs é a prática recomendada — o template sempre usa a AMI mais recente sem precisar ser atualizado. A desvantagem é que o deploy pode mudar sem alteração no template, o que complica o rastreamento de drift.

**Referenciando parâmetros:**

```yaml
Resources:
  MyInstance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref InstanceType   # !Ref retorna o valor do parâmetro
      ImageId: !Ref LatestAmiId
```

---

### 3. Resources — o coração do template

Cada recurso tem: logical ID (nome no template), Type e Properties.

```yaml
Resources:

  # Logical ID: nome interno — referenciado por Ref/GetAtt
  # Convenção: PascalCase descritivo (AppBucket, WebServerSG, etc.)
  AppBucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain             # Snapshot | Retain | Delete (padrão)
    UpdateReplacePolicy: Retain        # comportamento ao substituir o recurso
    DependsOn: LogGroup                # força ordem explícita (use só quando necessário)
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
        - !Ref WebServerSG          # Ref num recurso retorna o ID físico
      Tags:
        - Key: Name
          Value: !Sub "web-${Env}"
```

**`DeletionPolicy` e `UpdateReplacePolicy` — diferença crítica:**

```
DeletionPolicy       → o que acontece quando a STACK é deletada
UpdateReplacePolicy  → o que acontece quando o recurso precisa ser SUBSTITUÍDO
                       (ex: mudar o nome de um S3 bucket obriga substituição)

Valores:
  Delete    → recurso é deletado (padrão para a maioria)
  Retain    → recurso permanece na conta, desassociado da stack
  Snapshot  → tira snapshot antes de deletar (RDS, EBS, ElastiCache)
```

[FATO] `DeletionPolicy: Retain` num bucket S3 não impede que o CloudFormation tente deletar o bucket ao deletar a stack — apenas faz com que o CloudFormation "esqueça" o recurso em vez de deletá-lo. Se o bucket tiver objetos e não tiver `Retain`, o delete da stack vai falhar com `BucketNotEmpty`.

---

### 4. Ref e Fn::GetAtt — o que cada um retorna

Esta é a maior fonte de confusão em CloudFormation. `Ref` e `Fn::GetAtt` retornam coisas diferentes dependendo do tipo de recurso.

**`Ref` — retorna o identificador principal do recurso:**

```yaml
# Cada tipo de recurso tem um "Ref value" diferente:
!Ref AppBucket        # → nome do bucket (ex: "app-prod-123456789")
!Ref WebServerSG      # → Group ID (ex: "sg-0abc123")
!Ref WebServer        # → Instance ID (ex: "i-0abc123")
!Ref MyQueue          # → URL da fila (ex: "https://sqs.us-east-1...")
!Ref MyTable          # → nome da tabela DynamoDB
!Ref MyFunction       # → nome da função Lambda
!Ref MyRole           # → nome da role IAM (NÃO o ARN)
```

[FATO] Para saber o que `Ref` retorna para um tipo específico, consulte a documentação do recurso em "Return values > Ref". Não existe regra universal — cada serviço define o que é seu "identificador principal".

**`Fn::GetAtt` — retorna atributos específicos:**

```yaml
# Sintaxe longa
Fn::GetAtt:
  - AppBucket
  - Arn

# Sintaxe curta (YAML)
!GetAtt AppBucket.Arn           # → arn:aws:s3:::app-prod-123456789
!GetAtt AppBucket.DomainName    # → app-prod-123456789.s3.amazonaws.com
!GetAtt WebServerSG.GroupId     # → sg-0abc123 (mesmo que Ref aqui)
!GetAtt WebServer.PrivateIp     # → 10.0.1.50
!GetAtt WebServer.PublicIp      # → 3.x.x.x (se tiver IP público)
!GetAtt MyRole.Arn              # → arn:aws:iam::123:role/MyRole
!GetAtt MyFunction.Arn          # → arn:aws:lambda:us-east-1:123:function:name
```

**Diagrama da diferença:**

```
Recurso: AWS::S3::Bucket (logical ID: AppBucket)
         │
         ├── Ref           → nome do bucket   "app-prod-123456789012"
         ├── GetAtt .Arn   → ARN completo      "arn:aws:s3:::app-prod-..."
         └── GetAtt .DomainName → endpoint     "app-prod-....s3.amazonaws.com"

Recurso: AWS::IAM::Role (logical ID: MyRole)
         │
         ├── Ref           → nome da role      "MyRole-XXXXXXXXXXX"
         └── GetAtt .Arn   → ARN completo      "arn:aws:iam::123:role/MyRole-..."
         (para IAM Roles você quase sempre quer o ARN → use GetAtt, não Ref)
```

**Funções de string mais usadas junto com Ref/GetAtt:**

```yaml
# Sub — interpolação de strings (mais legível que Join)
!Sub "arn:aws:s3:::${AppBucket}/*"
!Sub "https://${WebServer.PublicIp}:443"
!Sub "${AWS::StackName}-${AWS::Region}-logs"   # pseudo-parâmetros

# Join — une lista com delimitador
!Join [",", [!Ref SubnetA, !Ref SubnetB]]

# Select — pega elemento de lista por índice
!Select [0, !GetAZs ""]   # primeira AZ da região atual

# Pseudo-parâmetros sempre disponíveis (sem declarar em Parameters):
# AWS::AccountId, AWS::Region, AWS::StackName, AWS::StackId, AWS::NoValue
```

---

### 5. Outputs e cross-stack references

Outputs servem para dois propósitos distintos: exibir valores úteis após o deploy, e exportar valores para outras stacks consumirem.

**Estrutura de um Output:**

```yaml
Outputs:

  BucketName:
    Description: "Nome do bucket de aplicação"
    Value: !Ref AppBucket          # valor exibido no console e retornado pela API

  BucketArn:
    Description: "ARN do bucket"
    Value: !GetAtt AppBucket.Arn
    Export:
      Name: !Sub "${AWS::StackName}-BucketArn"   # nome único na região/conta
```

**Consumindo um export em outra stack:**

```yaml
# Stack B importa o valor exportado pela Stack A
Resources:
  LambdaFunction:
    Type: AWS::Lambda::Function
    Properties:
      Environment:
        Variables:
          BUCKET_ARN: !ImportValue "minha-stack-infra-BucketArn"
```

**Restrições críticas do Export/ImportValue — [FATO]:**

```
1. Nomes de export são únicos por região/conta — não existem namespaces
2. Você NÃO pode deletar uma stack que tem outputs exportados
   enquanto outra stack estiver usando esses exports via ImportValue
3. Você NÃO pode modificar ou remover um output exportado
   enquanto ele estiver sendo importado
4. Exports NÃO funcionam cross-region — só cross-stack na mesma região
5. O nome do Export NÃO pode usar Ref/GetAtt de recursos
   (pode usar !Sub com pseudo-parâmetros como ${AWS::StackName})
```

[OPINIÃO — Alex Pulver, AWS CDK team] Para arquiteturas complexas, o acoplamento criado por Export/ImportValue torna operações como delete e refatoração difíceis. Uma alternativa é usar SSM Parameter Store para compartilhar valores entre stacks, mantendo o desacoplamento.

---

### 6. `aws cloudformation deploy` — o que acontece por baixo

`aws cloudformation deploy` é um wrapper de alto nível que:
1. Cria um changeset (nunca aplica direto)
2. Aguarda o changeset ser calculado
3. Executa o changeset automaticamente
4. Aguarda a stack atingir `CREATE_COMPLETE` ou `UPDATE_COMPLETE`

```bash
# Deploy básico
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name minha-stack \
  --parameter-overrides Env=prod InstanceType=t3.medium \
  --capabilities CAPABILITY_IAM \
  --profile prod

# Ver o changeset antes de executar (não aplica)
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name minha-stack \
  --no-execute-changeset

# Ver o changeset gerado
aws cloudformation describe-change-set \
  --stack-name minha-stack \
  --change-set-name <nome-do-changeset>

# Executar manualmente após revisar
aws cloudformation execute-change-set \
  --stack-name minha-stack \
  --change-set-name <nome-do-changeset>
```

**`--capabilities` — quando usar:**

```
CAPABILITY_IAM          → template cria roles ou policies IAM sem nome customizado
CAPABILITY_NAMED_IAM    → template cria roles com nome explícito (mais permissivo)
CAPABILITY_AUTO_EXPAND  → template usa Transform (SAM, macros customizadas)
```

[FATO] Sem a capability correta, o deploy falha com `InsufficientCapabilitiesException`. Isso é intencional — é uma proteção contra criação acidental de recursos IAM.

**Diagrama do fluxo de `aws cloudformation deploy`:**

```
aws cloudformation deploy
        │
        ▼
  Stack existe?
    │         │
   Não        Sim
    │          │
    ▼          ▼
CREATE_     UPDATE_
CHANGESET   CHANGESET
    │          │
    └────┬─────┘
         ▼
   Aguarda REVIEW_IN_PROGRESS
         │
         ▼
   Executa changeset
         │
         ▼
   Aguarda CREATE_COMPLETE
      ou UPDATE_COMPLETE
         │
   Erro? → ROLLBACK_COMPLETE
           (stack volta ao estado anterior)
```

**Status de stack mais importantes:**

```
CREATE_IN_PROGRESS    → criação em andamento
CREATE_COMPLETE       → criação concluída com sucesso
CREATE_FAILED         → falha na criação (stack permanece)
ROLLBACK_IN_PROGRESS  → rollback em andamento após falha
ROLLBACK_COMPLETE     → rollback concluído (stack existe mas vazia)
UPDATE_IN_PROGRESS    → atualização em andamento
UPDATE_COMPLETE       → atualização concluída
UPDATE_ROLLBACK_*     → rollback de atualização
DELETE_IN_PROGRESS    → deleção em andamento
DELETE_FAILED         → falha na deleção (recurso não pôde ser deletado)
```

---

## Exemplo prático

Template completo para uma aplicação com bucket S3 + role IAM + exportação de valores:

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: "Infraestrutura base — bucket de app + role de acesso"

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
    ConstraintDescription: "Lowercase, começa com letra, apenas letras/números/hifens"

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
    Description: "Nome do bucket"
    Value: !Ref AppBucket
    Export:
      Name: !Sub "${AWS::StackName}-BucketName"

  BucketArn:
    Description: "ARN do bucket"
    Value: !GetAtt AppBucket.Arn
    Export:
      Name: !Sub "${AWS::StackName}-BucketArn"

  AppRoleArn:
    Description: "ARN da role da aplicação"
    Value: !GetAtt AppRole.Arn
    Export:
      Name: !Sub "${AWS::StackName}-AppRoleArn"
```

**Deploy:**

```bash
# Validar o template antes de enviar
aws cloudformation validate-template --template-body file://template.yaml

# Deploy em dev
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name meu-projeto-infra \
  --parameter-overrides Env=dev ProjectName=meu-projeto \
  --capabilities CAPABILITY_NAMED_IAM \
  --profile dev

# Ver os outputs após o deploy
aws cloudformation describe-stacks \
  --stack-name meu-projeto-infra \
  --query 'Stacks[0].Outputs' \
  --output table

# Ver todos os exports disponíveis na conta/região
aws cloudformation list-exports --query 'Exports[*].[Name,Value]' --output table
```

---

## Armadilhas comuns

**1. Usar `Ref` em IAM Roles quando precisa do ARN**

`!Ref MyRole` retorna o nome da role, não o ARN. Quando você passa isso para um campo que espera ARN (como `RoleArn` de um ECS TaskDefinition), o CloudFormation aceita o template mas o serviço retorna erro em runtime. Sempre use `!GetAtt MyRole.Arn` para ARNs.

**2. Deletar uma stack com exports em uso**

Se outra stack usa `!ImportValue "minha-stack-BucketArn"`, você não pode deletar `minha-stack` nem modificar esse output. O CloudFormation retorna `Export cannot be updated as it is in use by another stack`. Para resolver: primeiro remova o `ImportValue` da stack consumidora, faça deploy dela, depois delete ou modifique a stack exportadora.

**3. `DependsOn` desnecessário**

Quando você usa `!Ref` ou `!GetAtt` de um recurso em outro, o CloudFormation já infere a dependência automaticamente. `DependsOn` explícito só é necessário quando um recurso depende de outro sem referenciar diretamente — por exemplo, uma função Lambda que consulta um endpoint que só existe após uma instância EC2 estar rodando.

---

## Exercício de reflexão

Você está migrando uma arquitetura existente para CloudFormation e precisa dividir a infra em duas stacks: `infra-base` (VPC, subnets, security groups) e `infra-app` (ECS service, ALB, roles IAM).

A stack `infra-app` precisa referenciar o VPC ID e os IDs das subnets criados pela `infra-base`. Um colega propõe usar `Export/ImportValue`. Outro propõe usar SSM Parameter Store como intermediário (a `infra-base` escreve os valores no SSM, a `infra-app` lê via `AWS::SSM::Parameter::Value`).

Quais são os trade-offs reais de cada abordagem? Considere: ciclo de vida das stacks, facilidade de debugging, operações de refatoração futura, e o que acontece quando você precisa recriar a `infra-base` do zero numa nova conta. Qual você escolheria e por quê?

---

## Recursos para aprofundar

**Anatomia e seções do template:**
- [CloudFormation template sections](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-anatomy.html) — referência completa de todas as seções com exemplos de cada campo.

**Funções intrínsecas:**
- [Intrinsic function reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/intrinsic-function-reference.html) — lista todas as funções com o que retornam por tipo de recurso. Consulte aqui sempre que tiver dúvida sobre o que `Ref` retorna para um recurso específico.

**Cross-stack references:**
- [Refer to resource outputs in another CloudFormation stack](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/walkthrough-crossstackref.html) — walkthrough completo de Export/ImportValue com os cenários de falha documentados.

**cfn-lint:**
- [cfn-lint no GitHub](https://github.com/aws-cloudformation/cfn-lint) — linter estático para templates CloudFormation. Detecta tipos errados, referências inválidas e propriedades inexistentes antes do deploy. Instale com `pip install cfn-lint` e rode com `cfn-lint template.yaml`.

---

