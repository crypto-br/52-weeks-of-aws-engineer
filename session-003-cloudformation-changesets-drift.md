# Sessão 003 — CloudFormation: changesets, drift detection e stack policies

**Duração estimada:** 60 minutos
**Pré-requisitos:** session-002 — CloudFormation: stacks, templates, parameters, outputs, Ref/GetAtt

---

## Objetivo

Ao final, você conseguirá criar e revisar um changeset antes de aplicar, detectar drift em recursos fora do ciclo CloudFormation, e configurar uma stack policy para proteger recursos críticos de substituição acidental.

---

## Contexto

[CONSENSO] O risco principal em operações de IaC em produção não é o deploy inicial — é a atualização. Um `aws cloudformation deploy` em produção pode deletar um banco de dados, substituir um security group com todas as suas regras, ou trocar o nome de uma fila SQS (perdendo mensagens em trânsito) se o engenheiro não entender o comportamento de update de cada tipo de recurso.

Esta sessão cobre as três ferramentas que existem para mitigar esse risco: changesets (para ver o que vai mudar antes de mudar), drift detection (para saber quando o mundo real divergiu do template), e stack policies (para impedir que certas mudanças aconteçam mesmo que sejam solicitadas).

[FATO] Changesets e stack policies são recursos do próprio CloudFormation. Drift detection é uma operação assíncrona que pode levar minutos em stacks grandes e não é executada automaticamente — você precisa acionar explicitamente.

---

## Conceitos principais

### 1. Update behaviors — o que pode acontecer com um recurso ao atualizar

Antes de entender changesets, é preciso entender o que o CloudFormation pode fazer com um recurso durante um update. Cada propriedade de cada tipo de recurso tem um "update behavior" documentado.

[FATO] Existem três categorias:

```
Update with No Interruption
  → CloudFormation atualiza o recurso sem interromper a operação
  → O recurso mantém seu physical ID
  → Ex: mudar tags de um EC2, mudar descrição de um Security Group

Update with Some Interruption
  → CloudFormation atualiza o recurso com interrupção temporária
  → O recurso mantém seu physical ID
  → Ex: mudar o tipo de instância de um EC2 (reboot necessário)

Replacement
  → CloudFormation cria um NOVO recurso, atualiza as referências,
    e deleta o recurso antigo
  → O recurso recebe um NOVO physical ID
  → Ex: mudar o nome de um bucket S3, mudar o engine de um RDS,
        mudar a VPC de um Security Group
```

**Por que Replacement é o mais perigoso:**

```
Estado anterior:                    Após Replacement:
  AppBucket (bucket-prod-123)  →     AppBucket (bucket-prod-456) [novo]
                                     bucket-prod-123 [deletado]

Se DeletionPolicy = Delete:  todos os objetos são perdidos
Se DeletionPolicy = Retain:  bucket antigo permanece órfão na conta
```

[FATO] Para identificar o update behavior de uma propriedade específica, consulte a documentação do recurso na coluna "Update requires" de cada propriedade. Não existe regra universal — depende do serviço e da propriedade.

**Diagrama dos update behaviors:**

```
Update da stack
      │
      ├─ Propriedade com "No Interruption"
      │         └─ Recurso atualizado in-place, sem impacto
      │
      ├─ Propriedade com "Some Interruption"
      │         └─ Recurso reiniciado, downtime mínimo
      │
      └─ Propriedade com "Replacement"
                └─ Novo recurso criado → referências atualizadas → antigo deletado
                   ⚠️ Physical ID muda, dados podem ser perdidos
```

---

### 2. Changesets — ver antes de fazer

Um changeset é um plano de execução que o CloudFormation calcula sem aplicar nada. Você cria, revisa e só então decide se executa.

**Fluxo completo via CLI:**

```bash
# 1. Criar o changeset (não aplica nada)
aws cloudformation create-change-set \
  --stack-name minha-stack \
  --change-set-name meu-changeset-$(date +%Y%m%d%H%M) \
  --template-body file://template.yaml \
  --parameters ParameterKey=Env,ParameterValue=prod \
  --capabilities CAPABILITY_NAMED_IAM

# 2. Aguardar o changeset ficar pronto (status: CREATE_COMPLETE)
aws cloudformation wait change-set-create-complete \
  --stack-name minha-stack \
  --change-set-name meu-changeset-20260506

# 3. Revisar o changeset
aws cloudformation describe-change-set \
  --stack-name minha-stack \
  --change-set-name meu-changeset-20260506 \
  --query 'Changes[*].ResourceChange.[Action,LogicalResourceId,ResourceType,Replacement]' \
  --output table

# 4a. Executar (se aprovado)
aws cloudformation execute-change-set \
  --stack-name minha-stack \
  --change-set-name meu-changeset-20260506

# 4b. Cancelar (se não aprovado)
aws cloudformation delete-change-set \
  --stack-name minha-stack \
  --change-set-name meu-changeset-20260506
```

**Lendo o output do describe-change-set:**

O campo mais importante é `Changes[].ResourceChange`. Cada item tem:

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

**O campo `Replacement` e seus valores:**

```
False        → nenhuma propriedade modificada exige substituição
True         → pelo menos uma propriedade exige substituição (certeza)
Conditional  → substituição depende de outras condições avaliadas em runtime
               (ex: quando uma propriedade muda para um valor específico)
```

[FATO] `Conditional` é o mais traiçoeiro — o CloudFormation não consegue determinar em tempo de planejamento se haverá substituição. Trate `Conditional` como `True` ao revisar changesets em produção.

**Comparando `aws cloudformation deploy` vs changeset manual:**

```
aws cloudformation deploy
  └── cria changeset internamente
  └── executa automaticamente
  └── você não vê o changeset antes da execução
  └── --no-execute-changeset cria o changeset mas NÃO executa

aws cloudformation create-change-set + execute-change-set
  └── fluxo explícito — você controla cada etapa
  └── recomendado em pipelines de produção com aprovação humana
```

**Múltiplos changesets numa stack:**

[FATO] Uma stack pode ter múltiplos changesets pendentes simultaneamente, mas só um pode ser executado — ao executar, o CloudFormation deleta todos os outros changesets da stack automaticamente (eles ficam obsoletos após a execução).

---

### 3. Drift detection — quando o mundo real diverge do template

Drift acontece quando alguém modifica um recurso fora do CloudFormation — via console, CLI direta, ou outra ferramenta. O template diz uma coisa, o recurso está em outro estado.

**Acionando drift detection:**

```bash
# Detectar drift em toda a stack (operação assíncrona)
aws cloudformation detect-stack-drift \
  --stack-name minha-stack
# Retorna: { "StackDriftDetectionId": "abc-123" }

# Acompanhar o status da detecção
aws cloudformation describe-stack-drift-detection-status \
  --stack-drift-detection-id abc-123
# DetectionStatus: DETECTION_IN_PROGRESS | DETECTION_COMPLETE | DETECTION_FAILED

# Ver o resultado por recurso
aws cloudformation describe-stack-resource-drifts \
  --stack-name minha-stack \
  --stack-resource-drift-filter-status DRIFTED \
  --query 'StackResourceDrifts[*].[LogicalResourceId,ResourceType,StackResourceDriftStatus]' \
  --output table

# Detectar drift num recurso específico
aws cloudformation detect-stack-resource-drift \
  --stack-name minha-stack \
  --logical-resource-id AppBucket
```

**Status de drift por recurso:**

```
IN_SYNC     → propriedades conferem com o template
DRIFTED     → pelo menos uma propriedade difere do template
NOT_CHECKED → recurso não suporta drift detection ou não foi verificado
DELETED     → recurso foi deletado fora do CloudFormation
```

**O que o drift detection verifica:**

[FATO] O CloudFormation compara apenas as propriedades **explicitamente definidas no template** com o estado atual do recurso. Propriedades não declaradas no template (que ficam com valores default do serviço) não são verificadas.

```yaml
# Template declara apenas BucketName e VersioningConfiguration
AppBucket:
  Type: AWS::S3::Bucket
  Properties:
    BucketName: meu-bucket
    VersioningConfiguration:
      Status: Enabled

# Alguém adicionou uma política de lifecycle via console
# → drift detection VAI detectar o lifecycle como "added" (não estava no template)
# → drift detection NÃO vai reportar a ausência de CORS (nunca foi declarado)
```

**Limitações importantes do drift detection — [FATO]:**

```
1. Nem todos os tipos de recurso suportam drift detection
   (verifique a lista em "Resources that support import and drift detection operations")

2. A detecção é assíncrona e pode levar vários minutos em stacks grandes

3. Drift detection NÃO corrige o drift — apenas reporta
   Para corrigir: ou atualiza o recurso manualmente para o estado do template,
   ou atualiza o template para refletir o estado atual

4. Não existe drift detection contínua nativa — você precisa agendar via EventBridge
   ou usar AWS Config com a regra `cloudformation-stack-drift-detection-check`

5. Stacks com status REVIEW_IN_PROGRESS não podem ter drift detectado
```

**Diagrama do ciclo de drift:**

```
Template                     Recurso Real
─────────                    ────────────
VersioningConfiguration: Enabled
                             VersioningConfiguration: Enabled   ← IN_SYNC

[Alguém vai no console e desativa o versioning]

VersioningConfiguration: Enabled
                             VersioningConfiguration: Suspended ← DRIFTED

Opções para resolver:
  A) Reativar o versioning no console/CLI → IN_SYNC
  B) Atualizar o template para Suspended e fazer deploy → IN_SYNC
  C) Fazer deploy do template original (sem mudar nada) → CloudFormation
     detecta a diferença e reativa o versioning como parte do update
```

---

### 4. Stack policies — proteção declarativa contra atualizações destrutivas

Uma stack policy é um documento JSON que define quais ações de update são permitidas em quais recursos. Uma vez definida, toda atualização da stack passa pela policy antes de ser executada.

**Estrutura da policy:**

```json
{
  "Statement": [
    {
      "Effect": "Allow" | "Deny",
      "Principal": "*",
      "Action": [ "Update:Modify", "Update:Replace", "Update:Delete", "Update:*" ],
      "Resource": "LogicalResourceId/NomeDoRecurso" | "*",
      "Condition": { ... }   // opcional
    }
  ]
}
```

**As quatro Actions possíveis:**

```
Update:Modify    → atualização que não substitui nem deleta
Update:Replace   → atualização que substitui o recurso (Replacement = True)
Update:Delete    → remoção do recurso do template
Update:*         → qualquer atualização (wildcard)
```

**Comportamento padrão — [FATO]:**

Sem stack policy: todos os recursos podem ser atualizados sem restrição.
Com stack policy: todos os recursos ficam protegidos por padrão (`Deny Update:*`). Você precisa declarar explicitamente o que é permitido.

**Exemplo prático — proteger RDS e SQS, liberar o resto:**

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

[FATO] Deny sempre prevalece sobre Allow quando há sobreposição de statements. A lógica é idêntica ao IAM.

**Aplicando e gerenciando a policy:**

```bash
# Aplicar a policy ao criar a stack
aws cloudformation create-stack \
  --stack-name minha-stack \
  --template-body file://template.yaml \
  --stack-policy-body file://stack-policy.json

# Aplicar a uma stack existente
aws cloudformation set-stack-policy \
  --stack-name minha-stack \
  --stack-policy-body file://stack-policy.json

# Ver a policy atual
aws cloudformation get-stack-policy \
  --stack-name minha-stack

# Override temporário para atualizar um recurso protegido
# (a policy original volta a valer automaticamente após o update)
aws cloudformation update-stack \
  --stack-name minha-stack \
  --template-body file://template.yaml \
  --stack-policy-during-update-body file://override-policy.json
```

**Override temporário — política de emergência:**

```json
// override-policy.json — permite substituição apenas do RDS
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

[FATO] O override via `--stack-policy-during-update-body` é aplicado apenas durante aquele update específico. Após a conclusão do update (sucesso ou rollback), a policy original é restaurada automaticamente.

---

### 5. Stack termination protection — complemento às stack policies

Stack policy protege recursos de updates. Termination protection protege a stack inteira de ser deletada.

```bash
# Habilitar
aws cloudformation update-termination-protection \
  --stack-name minha-stack \
  --enable-termination-protection

# Verificar
aws cloudformation describe-stacks \
  --stack-name minha-stack \
  --query 'Stacks[0].EnableTerminationProtection'

# Desabilitar (necessário antes de deletar)
aws cloudformation update-termination-protection \
  --stack-name minha-stack \
  --no-enable-termination-protection
```

**Diferença entre as proteções:**

```
Termination protection  → impede DELETE da stack inteira
Stack policy            → impede UPDATE destrutivo de recursos específicos
DeletionPolicy: Retain  → comportamento do recurso quando a stack é deletada
```

As três são complementares e independentes.

---

## Exemplo prático

**Cenário:** você tem uma stack de produção com um RDS e precisa atualizar o tipo de instância — uma operação de `Some Interruption`. Você quer garantir que não vai acidentalmente substituir o banco.

```bash
# 1. Ver o estado atual da stack
aws cloudformation describe-stacks \
  --stack-name prod-stack \
  --query 'Stacks[0].[StackStatus,EnableTerminationProtection]'

# 2. Verificar se há drift antes de qualquer operação
aws cloudformation detect-stack-drift --stack-name prod-stack
# aguardar...
aws cloudformation describe-stack-resource-drifts \
  --stack-name prod-stack \
  --stack-resource-drift-filter-status DRIFTED \
  --output table
# Se houver drift, avaliar antes de prosseguir

# 3. Criar changeset com a mudança de instance class
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

# 4. Revisar: verificar se Replacement é False (esperado para mudança de class)
aws cloudformation describe-change-set \
  --stack-name prod-stack \
  --change-set-name update-rds-class-20260506 \
  --query 'Changes[*].ResourceChange.[Action,LogicalResourceId,Replacement,Scope]' \
  --output table

# Saída esperada:
# Action   | LogicalResourceId | Replacement | Scope
# Modify   | ProductionDB      | False       | ['Properties']
# ✅ Replacement = False → seguro prosseguir

# 5. Executar após confirmação
aws cloudformation execute-change-set \
  --stack-name prod-stack \
  --change-set-name update-rds-class-20260506

# 6. Acompanhar eventos em tempo real
aws cloudformation describe-stack-events \
  --stack-name prod-stack \
  --query 'StackEvents[0:10].[Timestamp,LogicalResourceId,ResourceStatus,ResourceStatusReason]' \
  --output table
```

---

## Armadilhas comuns

**1. Confiar em `Replacement: False` sem verificar `RequiresRecreation` nos Details**

O campo `Replacement` no nível do resource change é um agregado. Para entender qual propriedade específica está causando a mudança — e se ela poderia causar replacement em cenários diferentes — você precisa olhar `Changes[].ResourceChange.Details[].Target.RequiresRecreation`. Um changeset pode mostrar `Replacement: False` hoje e `Replacement: True` amanhã se o valor da propriedade mudar.

**2. Achar que drift detection cobre todos os recursos**

Vários tipos de recurso não suportam drift detection — entre eles alguns recursos Lambda, recursos de serviços mais novos, e recursos customizados (`AWS::CloudFormation::CustomResource`). Antes de confiar num relatório de drift limpo, verifique quais recursos da stack estão marcados como `NOT_CHECKED` no output.

**3. Stack policy bloqueando o próprio rollback**

[FATO] Se uma stack policy nega `Update:Replace` num recurso, e um update falha e tenta fazer rollback criando uma nova versão do recurso, o rollback também pode ser bloqueado pela policy — deixando a stack em `UPDATE_ROLLBACK_FAILED`. Para sair desse estado: `aws cloudformation continue-update-rollback` com `--resources-to-skip` para pular o recurso problemático.

---

## Exercício de reflexão

Você mantém uma stack de produção com os seguintes recursos: um RDS PostgreSQL, uma fila SQS, um bucket S3 e um ALB. Um colega cria uma stack policy que apenas nega `Update:Replace` e `Update:Delete` para o RDS e a fila SQS, e permite `Update:*` para tudo mais.

Três meses depois, alguém muda o `BucketName` do S3 no template (uma operação de Replacement) e faz deploy. O bucket é substituído e os dados são perdidos.

Onde a proteção falhou? Reescreva a stack policy para cobrir esse caso sem bloquear operações legítimas de manutenção nos outros recursos. Considere também: o que precisaria ser feito no template além da stack policy para garantir proteção em múltiplas camadas?

---

## Recursos para aprofundar

**Changesets:**
- [Update CloudFormation stacks using change sets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-changesets.html) — cobre o ciclo completo: criação, visualização e execução, com os campos do output explicados.

**Update behaviors:**
- [Understand update behaviors of stack resources](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-update-behaviors.html) — referência dos três comportamentos com exemplos de recursos e propriedades para cada categoria.

**Drift detection:**
- [Detect unmanaged configuration changes to stacks and resources with drift detection](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-stack-drift.html) — inclui a lista de recursos suportados e o formato detalhado do output de drift por recurso.

**Stack policies:**
- [Prevent updates to stack resources](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/protect-stack-resources.html) — estrutura da policy JSON, as quatro actions disponíveis, e como fazer override temporário.

---

