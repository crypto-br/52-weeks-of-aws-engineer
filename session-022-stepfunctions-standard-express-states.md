# Sessão session-022 — Step Functions: Standard vs Express, states básicos (Task, Choice, Wait) e execução

**Duração estimada:** 60 minutos
**Pré-requisitos:** session-021-lambda-extensions-layers-powertools

---

## Objetivo

Ao final, você conseguirá criar uma state machine com Task, Choice, Wait, Succeed e Fail states, executá-la via console e CLI, entender as garantias de execução e custo de Standard vs Express Workflows, e ler um execution history para identificar onde uma execução falhou.

---

## Contexto

[FATO] AWS Step Functions é um serviço de orquestração de workflows serverless lançado em 2016. Diferente de soluções de mensageria (SQS, SNS) que apenas transportam dados, Step Functions coordena *sequências de estados* — incluindo lógica condicional, esperas, paralelismo e tratamento de erros — de forma declarativa, durável e auditável.

[CONSENSO] O valor central do serviço está em tirar do código da aplicação a responsabilidade de orquestrar fluxos de longa duração ou com múltiplas etapas. Sem Step Functions, esse estado é frequentemente gerenciado via flags em banco de dados, filas encadeadas ou lógica de retry embutida em funções Lambda — padrões difíceis de monitorar, testar e depurar. Step Functions externaliza esse estado para o próprio serviço, tornando o fluxo visível, reproduzível e auditável no console.

[FATO] A linguagem usada para definir state machines é a **Amazon States Language (ASL)**, um formato JSON descrito em especificação aberta em [states-language.net](https://states-language.net/spec.html). A ASL define oito tipos de estados: Task, Choice, Wait, Succeed, Fail, Pass, Parallel e Map.

---

## Conceitos principais

### 1. Standard Workflows vs Express Workflows

Existem dois modos de operação radicalmente diferentes em Step Functions. A escolha entre eles afeta garantias de execução, auditabilidade, duração máxima e modelo de preço.

[FATO] A tabela abaixo resume as diferenças documentadas na AWS:

```
┌─────────────────────────┬──────────────────────────┬──────────────────────────────────┐
│ Dimensão                │ Standard Workflow         │ Express Workflow                  │
├─────────────────────────┼──────────────────────────┼──────────────────────────────────┤
│ Garantia de execução    │ Exactly-once             │ Async: at-least-once             │
│                         │                          │ Sync:  at-most-once              │
├─────────────────────────┼──────────────────────────┼──────────────────────────────────┤
│ Duração máxima          │ 1 ano                    │ 5 minutos                         │
├─────────────────────────┼──────────────────────────┼──────────────────────────────────┤
│ Throughput              │ > 2.000 exec/s           │ > 100.000 exec/s                  │
├─────────────────────────┼──────────────────────────┼──────────────────────────────────┤
│ Execution history       │ GetExecutionHistory API  │ CloudWatch Logs (obrigatório)     │
├─────────────────────────┼──────────────────────────┼──────────────────────────────────┤
│ Modelo de preço         │ Por state transition      │ Por execução + GB-segundo         │
│                         │ $0.025 / 1.000 trans.    │ $1.00 / 1M exec + $0.00001/GB·s  │
├─────────────────────────┼──────────────────────────┼──────────────────────────────────┤
│ Caso de uso ideal       │ Workflows de negócio,     │ Alta volumetria, event-driven,   │
│                         │ processos duráveis        │ IoT, streaming, microserviços    │
└─────────────────────────┴──────────────────────────┴──────────────────────────────────┘
```

[FATO] **Exactly-once** em Standard Workflows significa que cada estado é iniciado no máximo uma vez, a menos que você configure explicitamente `Retry` no estado. O serviço garante isso durando o estado persistido em seu backend, mesmo que haja falhas internas na infraestrutura da AWS.

[FATO] **At-least-once** em Express Workflows assíncronos significa que a execução *pode* iniciar mais de uma vez em cenários de falha interna. Isso implica que os serviços chamados devem ser **idempotentes** — o mesmo input executado duas vezes não pode produzir efeitos colaterais distintos.

[CONSENSO] A regra de bolso mais usada pela comunidade: se o workflow envolve transações financeiras, aprovações humanas, ou qualquer operação não-idempotente com duração > 5 min → Standard. Se envolve processamento de eventos em alta frequência onde idempotência é natural → Express.

**Cálculo de custo comparativo (exemplo):**

Cenário: 10.000 execuções/dia, 5 estados por execução, 30 dias.
- Standard: 10.000 × 5 × 30 = 1.500.000 transitions × $0.025/1.000 = **$37,50/mês**
- Express (100ms, 64MB): 300.000 exec × $1.00/1M + 300.000 × 0.1s × 64MB/1024 × $0.00001 = $0.30 + **$0.019 ≈ $0.32/mês**

[INCERTO] Os preços acima refletem a documentação da AWS em maio de 2026. Verifique sempre a página de pricing atualizada, pois Express Workflows tiveram ajustes de preço nos últimos anos.

---

### 2. Amazon States Language (ASL) — estrutura de uma state machine

[FATO] Uma definição ASL é um objeto JSON com os seguintes campos obrigatórios:

```json
{
  "Comment": "Descrição opcional",
  "StartAt": "NomePrimeiroEstado",
  "States": {
    "NomePrimeiroEstado": { ... },
    "OutroEstado": { ... }
  }
}
```

`StartAt` aponta para o nome de um estado dentro de `States`. Cada estado deve ter um campo `Type` e, para estados não-terminais, um campo `Next` (ou `End: true` para o estado final).

```
Estado inicial                Transição                    Estado terminal
──────────────────────────────────────────────────────────────────────────────
{                                                          {
  "Type": "Task",       ──── "Next": "ProximoEstado" ────   "Type": "Succeed"
  "Resource": "...",                                        (ou "Fail")
  "Next": "ProximoEstado"                                 }
}
```

[FATO] Regras de transição:
- Estados com `"End": true` são terminais — a execução encerra neles.
- `Succeed` e `Fail` são sempre terminais (não têm `Next`).
- `Choice` é o único tipo que não tem `Next` no nível superior — ele define `Choices[]` com `Next` em cada regra.
- Uma execução *falha* se chegar a um estado `Fail` ou se ocorrer um erro não tratado.
- Uma execução *sucede* se chegar a `Succeed` ou a qualquer estado com `"End": true`.

---

### 3. Task State — invocando serviços e aguardando resultados

[FATO] O estado `Task` é o coração de qualquer workflow: ele representa uma unidade de trabalho delegada a um serviço externo. O campo `Resource` é um ARN que especifica o que será invocado.

Três padrões de integração distintos:

```
PADRÃO 1 — Request-Response (default)
───────────────────────────────────────────────────────────────────────────────
Step Functions invoca o serviço e avança IMEDIATAMENTE após o aceite da chamada.
Não aguarda o resultado. Adequado quando o resultado não importa ou é assíncrono.

"Resource": "arn:aws:states:::lambda:invoke"


PADRÃO 2 — Synchronous (sufixo .sync:2)
───────────────────────────────────────────────────────────────────────────────
Step Functions aguarda a conclusão do job/operação antes de avançar.
AWS SDK Integrations usam polling interno via CloudWatch Events.

"Resource": "arn:aws:states:::lambda:invoke.waitForTaskToken"
             arn:aws:states:::dynamodb:putItem.sync:2
             arn:aws:states:::ecs:runTask.sync:2
             arn:aws:states:::glue:startJobRun.sync

PADRÃO 3 — Wait for Task Token
───────────────────────────────────────────────────────────────────────────────
Step Functions para e aguarda até que um worker externo chame
SendTaskSuccess ou SendTaskFailure com o token recebido.
Adequado para aprovações humanas, processos legados, callbacks.

"Resource": "arn:aws:states:::sqs:sendMessage.waitForTaskToken"
```

[FATO] Exemplo de Task invocando Lambda com `.waitForTaskToken`:

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
  "Next": "ProcessarResposta"
}
```

`$$.Task.Token` é uma referência especial ao **Context Object** — dados sobre a execução atual que Step Functions injeta. O prefixo `$$` acessa o contexto; `$` acessa o input do estado.

[FATO] Campos opcionais importantes no Task State:

| Campo | Descrição |
|---|---|
| `TimeoutSeconds` | Tempo máximo de espera antes de lançar `States.Timeout` |
| `HeartbeatSeconds` | Intervalo máximo entre heartbeats; se excedido, lança `States.HeartbeatTimeout` |
| `Retry` | Array de regras de retry com `ErrorEquals`, `IntervalSeconds`, `MaxAttempts`, `BackoffRate` |
| `Catch` | Array de fallbacks: se erro em `ErrorEquals`, transiciona para estado `Next` |
| `ResultPath` | Onde no input gravar o resultado (ex: `"$.resultado"` para não sobrescrever input) |
| `Parameters` | Reformata o input antes de passar ao serviço (suporta `.$` para referências) |
| `ResultSelector` | Filtra/reformata a saída do serviço antes de gravar |

---

### 4. Choice State — lógica condicional

[FATO] O `Choice` state avalia uma lista de regras em ordem e transiciona para o primeiro `Next` cujas condições sejam verdadeiras. Se nenhuma regra casar, usa `Default` (obrigatório na prática para evitar execução falhando).

```json
"VerificarStatus": {
  "Type": "Choice",
  "Choices": [
    {
      "Variable": "$.status",
      "StringEquals": "APROVADO",
      "Next": "ProcessarAprovado"
    },
    {
      "Variable": "$.tentativas",
      "NumericGreaterThanEquals": 3,
      "Next": "EscalarParaHumano"
    },
    {
      "And": [
        { "Variable": "$.valor", "NumericGreaterThan": 1000 },
        { "Variable": "$.categoria", "StringEquals": "VIP" }
      ],
      "Next": "ProcessarVIP"
    }
  ],
  "Default": "ProcessarPadrao"
}
```

[FATO] Operadores de comparação disponíveis na ASL (seleção):

```
StringEquals / StringEqualsPath
StringLessThan / StringGreaterThan / StringMatches (glob, ex: "foo*")
NumericEquals / NumericLessThan / NumericGreaterThan / ...
BooleanEquals / BooleanEqualsPath
TimestampEquals / TimestampLessThan / TimestampGreaterThan / ...
IsNull / IsPresent / IsString / IsNumeric / IsBoolean / IsTimestamp
And / Or / Not  (combinadores lógicos)
```

[CONSENSO] A sufixo `Path` (ex: `StringEqualsPath`) permite comparar o valor da `Variable` com o valor de *outra path* no input — útil quando o valor de comparação também vem do input dinâmico.

```
IMPORTANTE: Choice não tem "Next" no nível superior.
Cada regra dentro de "Choices[]" tem seu próprio "Next".
Choice também não pode ter "End: true".
```

---

### 5. Wait State — pausas temporais

[FATO] O `Wait` state pausa a execução sem consumir recursos de computação. O billing de Standard Workflow continua (estado transitou ao entrar em Wait), mas nenhum Lambda ou serviço externo fica rodando.

Quatro formas de especificar a duração:

```json
// 1. Duração fixa em segundos
{
  "Type": "Wait",
  "Seconds": 300,
  "Next": "ProximoEstado"
}

// 2. Timestamp absoluto fixo
{
  "Type": "Wait",
  "Timestamp": "2026-12-31T23:59:59Z",
  "Next": "ProximoEstado"
}

// 3. Duração vinda do input (JsonPath)
{
  "Type": "Wait",
  "SecondsPath": "$.aguardar_segundos",
  "Next": "ProximoEstado"
}

// 4. Timestamp vindo do input (JsonPath)
{
  "Type": "Wait",
  "TimestampPath": "$.data_processamento",
  "Next": "ProximoEstado"
}
```

[CONSENSO] `TimestampPath` é o padrão mais poderoso: permite agendar o prosseguimento da execução para um instante específico calculado em estados anteriores. Exemplo: enviar um lembrete 24h após o usuário criar um pedido, usando o timestamp gerado pela Lambda de criação.

---

### 6. Succeed e Fail States — estados terminais

[FATO] `Succeed` encerra a execução com status `SUCCEEDED`. Não tem `Next`, não tem campos obrigatórios além de `Type`.

```json
"Finalizado": {
  "Type": "Succeed"
}
```

[FATO] `Fail` encerra a execução com status `FAILED`. Aceita `Error` e `Cause` para descrever o motivo:

```json
"ErroNegocio": {
  "Type": "Fail",
  "Error": "PedidoInvalido",
  "Cause": "Valor do pedido excede limite de crédito disponível"
}
```

[FATO] Também é possível usar `ErrorPath` e `CausePath` (JsonPath) para preencher esses campos dinamicamente a partir do input do estado — útil quando o Catch de um Task repassa o erro para um estado Fail descritivo.

---

### 7. Execution History — lendo e depurando execuções

[FATO] Para Standard Workflows, a API `GetExecutionHistory` retorna todos os eventos de uma execução em ordem cronológica. Cada evento tem `type`, `timestamp`, e payload específico do tipo.

Tipos de eventos mais relevantes para diagnóstico:

```
ExecutionStarted           — input inicial da execução
TaskStateEntered           — quando um Task state foi iniciado
TaskScheduled              — quando o serviço externo foi invocado
TaskStarted                — confirmação que o serviço começou
TaskSucceeded              — resultado bem-sucedido do serviço
TaskFailed                 — erro retornado pelo serviço
TaskTimedOut               — TimeoutSeconds ou HeartbeatSeconds excedido
ChoiceStateEntered         — Choice state sendo avaliado
WaitStateEntered/Exited    — início e fim de espera
ExecutionFailed            — execução encerrou com falha (inclui Error e Cause)
ExecutionSucceeded         — execução encerrou com sucesso
```

[FATO] Para Express Workflows, `GetExecutionHistory` **não está disponível**. Os eventos devem ser enviados para CloudWatch Logs, configurando `logging_configuration` com `level` `ALL`, `ERROR`, `FATAL` ou `OFF`. A análise é feita via CloudWatch Log Insights.

---

## Exemplo prático

**Cenário:** Workflow de processamento de pedidos de e-commerce. Um pedido entra, é validado por uma Lambda, aguarda 2 horas para antifraude, é aprovado ou rejeitado via Choice, e finaliza.

### Definição ASL completa

```json
{
  "Comment": "Workflow de processamento de pedido",
  "StartAt": "ValidarPedido",
  "States": {

    "ValidarPedido": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "arn:aws:lambda:us-east-1:123456789:function:ValidarPedido",
        "Payload.$": "$"
      },
      "ResultSelector": {
        "valido.$": "$.Payload.valido",
        "score_fraude.$": "$.Payload.score_fraude",
        "pedido_id.$": "$.Payload.pedido_id"
      },
      "ResultPath": "$.validacao",
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
          "ResultPath": "$.erro",
          "Next": "FalhaValidacao"
        }
      ],
      "Next": "AguardarAntifraude"
    },

    "AguardarAntifraude": {
      "Type": "Wait",
      "Seconds": 7200,
      "Next": "VerificarFraude"
    },

    "VerificarFraude": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.validacao.valido",
          "BooleanEquals": false,
          "Next": "PedidoRejeitado"
        },
        {
          "Variable": "$.validacao.score_fraude",
          "NumericGreaterThan": 80,
          "Next": "PedidoSuspeito"
        }
      ],
      "Default": "AprovarPedido"
    },

    "AprovarPedido": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "arn:aws:lambda:us-east-1:123456789:function:AprovarPedido",
        "Payload.$": "$"
      },
      "ResultPath": null,
      "Next": "PedidoAprovado"
    },

    "PedidoAprovado": {
      "Type": "Succeed"
    },

    "PedidoSuspeito": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sqs:sendMessage.waitForTaskToken",
      "Parameters": {
        "QueueUrl": "https://sqs.us-east-1.amazonaws.com/123/revisao-manual",
        "MessageBody": {
          "TaskToken.$": "$$.Task.Token",
          "PedidoId.$": "$.validacao.pedido_id",
          "ScoreFraude.$": "$.validacao.score_fraude"
        }
      },
      "TimeoutSeconds": 86400,
      "Next": "AprovarPedido"
    },

    "PedidoRejeitado": {
      "Type": "Fail",
      "Error": "PedidoInvalido",
      "Cause": "Pedido rejeitado na validação ou análise de fraude"
    },

    "FalhaValidacao": {
      "Type": "Fail",
      "Error": "ErroValidacao",
      "CausePath": "$.erro.Cause"
    }

  }
}
```

### Diagrama do fluxo

```
                   ┌──────────────┐
                   │ ValidarPedido│ ◄─── Lambda invoke + Retry
                   └──────┬───────┘
                          │ ResultPath: $.validacao
                          ▼
               ┌──────────────────────┐
               │ AguardarAntifraude   │ (Wait 2h)
               └──────────┬───────────┘
                          ▼
               ┌──────────────────────┐
               │   VerificarFraude    │ (Choice)
               └──┬──────────┬────────┘
                  │          │           └──────────────────────┐
             valido=false  score>80       (default)             │
                  │          │                                   │
                  ▼          ▼                                   ▼
         ┌──────────┐  ┌───────────────┐              ┌──────────────────┐
         │ Pedido   │  │ PedidoSuspeito│ (waitForToken)│  AprovarPedido  │
         │ Rejeitado│  │ → SQS → Manual│──────────────►│  Lambda invoke  │
         │ (Fail)   │  └───────────────┘              └────────┬─────────┘
         └──────────┘                                          ▼
                                                     ┌──────────────────┐
                                                     │  PedidoAprovado  │
                                                     │   (Succeed)      │
                                                     └──────────────────┘
```

### CDK — criando a state machine

```python
from aws_cdk import (
    Stack,
    aws_stepfunctions as sfn,
    aws_stepfunctions_tasks as tasks,
    aws_lambda as lambda_,
    Duration,
)
import json

class PedidoWorkflowStack(Stack):

    def __init__(self, scope, construct_id, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # Funções Lambda (já existentes ou definidas aqui)
        fn_validar = lambda_.Function.from_function_name(
            self, "FnValidar", "ValidarPedido"
        )
        fn_aprovar = lambda_.Function.from_function_name(
            self, "FnAprovar", "AprovarPedido"
        )

        # ── Estados ──────────────────────────────────────────────────
        falha_validacao = sfn.Fail(
            self, "FalhaValidacao",
            error="ErroValidacao",
            cause="Erro ao validar pedido",
        )

        pedido_rejeitado = sfn.Fail(
            self, "PedidoRejeitado",
            error="PedidoInvalido",
            cause="Rejeitado na análise de fraude",
        )

        pedido_aprovado = sfn.Succeed(self, "PedidoAprovado")

        aprovar_pedido = tasks.LambdaInvoke(
            self, "AprovarPedido",
            lambda_function=fn_aprovar,
            result_path=sfn.JsonPath.DISCARD,  # ResultPath: null
        ).next(pedido_aprovado)

        verificar_fraude = sfn.Choice(self, "VerificarFraude") \
            .when(
                sfn.Condition.boolean_equals("$.validacao.valido", False),
                pedido_rejeitado
            ) \
            .when(
                sfn.Condition.number_greater_than("$.validacao.score_fraude", 80),
                # PedidoSuspeito simplificado para o exemplo CDK
                aprovar_pedido
            ) \
            .otherwise(aprovar_pedido)

        aguardar_antifraude = sfn.Wait(
            self, "AguardarAntifraude",
            time=sfn.WaitTime.duration(Duration.hours(2)),
        ).next(verificar_fraude)

        validar_pedido = tasks.LambdaInvoke(
            self, "ValidarPedido",
            lambda_function=fn_validar,
            result_selector={
                "valido": sfn.JsonPath.string_at("$.Payload.valido"),
                "score_fraude": sfn.JsonPath.number_at("$.Payload.score_fraude"),
                "pedido_id": sfn.JsonPath.string_at("$.Payload.pedido_id"),
            },
            result_path="$.validacao",
            timeout=Duration.seconds(30),
        ).add_retry(
            errors=["Lambda.ServiceException", "Lambda.TooManyRequestsException"],
            interval=Duration.seconds(2),
            max_attempts=3,
            backoff_rate=2,
        ).add_catch(
            falha_validacao,
            errors=["States.ALL"],
            result_path="$.erro",
        ).next(aguardar_antifraude)

        # ── State Machine ─────────────────────────────────────────────
        state_machine = sfn.StateMachine(
            self, "ProcessamentoPedido",
            state_machine_name="ProcessamentoPedido",
            definition_body=sfn.DefinitionBody.from_chainable(validar_pedido),
            # Para Standard Workflow (default):
            state_machine_type=sfn.StateMachineType.STANDARD,
        )
```

### Executar e inspecionar via CLI

```bash
# Iniciar execução
aws stepfunctions start-execution \
  --state-machine-arn arn:aws:states:us-east-1:123456789:stateMachine:ProcessamentoPedido \
  --name "exec-pedido-001" \
  --input '{"pedido_id": "P001", "valor": 1500, "cliente_id": "C42"}'

# Listar execuções recentes
aws stepfunctions list-executions \
  --state-machine-arn arn:aws:states:us-east-1:123456789:stateMachine:ProcessamentoPedido \
  --status-filter RUNNING

# Descrever uma execução (status + input/output)
aws stepfunctions describe-execution \
  --execution-arn arn:aws:states:us-east-1:123456789:execution:ProcessamentoPedido:exec-pedido-001

# Ver histórico completo (Standard Workflow apenas)
aws stepfunctions get-execution-history \
  --execution-arn arn:aws:states:us-east-1:123456789:execution:ProcessamentoPedido:exec-pedido-001 \
  --query 'events[?type==`TaskFailed` || type==`ExecutionFailed`]'
```

---

## Armadilhas comuns

### Armadilha 1 — Confundir `ResultPath: null` com ausência de `ResultPath`

**O erro:** O desenvolvedor quer que o output do Task não altere o estado atual e simplesmente omite `ResultPath`. O resultado é que o output do serviço **substitui completamente** o input do estado — todos os campos anteriores são perdidos.

**Por que acontece:** O comportamento padrão de `ResultPath` quando ausente é `$` — ou seja, o resultado do serviço se torna o novo input do próximo estado, sobrescrevendo tudo.

**Como evitar:**
- `ResultPath: null` → descarta o resultado, passa o input original adiante.
- `ResultPath: "$.resultado"` → preserva o input e **adiciona** o resultado no campo `.resultado`.
- `ResultPath: "$"` (default) → substitui o input inteiro pelo resultado.

```json
// Errado: sem ResultPath, o output da Lambda substitui $.pedido_id, $.valor etc.
"AprovarPedido": { "Type": "Task", "Resource": "...", "Next": "..." }

// Correto: descarta resultado, preserva input
"AprovarPedido": { "Type": "Task", "Resource": "...", "ResultPath": null, "Next": "..." }
```

---

### Armadilha 2 — Usar Express Workflow onde o Task não é idempotente

**O erro:** O desenvolvedor escolhe Express Workflow (mais barato, maior throughput) para um workflow que inclui uma Task que cria um recurso único (ex: inserir registro em banco com ID auto-incremento). Em caso de retry interno do serviço (at-least-once), o mesmo registro é criado duas vezes.

**Por que acontece:** Express Workflows assíncronos têm garantia at-least-once, não exactly-once. O desenvolvedor frequentemente não percebe que "at-least-once" significa que a execução *pode* rodar novamente.

**Como evitar:**
- Para workflows com Tasks não-idempotentes: use Standard Workflow.
- Se quiser usar Express por custo, garanta que todas as Tasks sejam idempotentes (upsert em vez de insert, chaves únicas via `$.input.id`, etc.).
- Express Workflow **síncrono** tem garantia at-most-once — mas isso significa que pode *não executar* em falhas, não que executará exatamente uma vez.

---

### Armadilha 3 — Esquecer o `TimeoutSeconds` em Tasks com `.waitForTaskToken`

**O erro:** O estado `WaitForTaskToken` é configurado sem `TimeoutSeconds`. O workflow fica bloqueado indefinidamente se o worker que deveria enviar o token falhar silenciosamente (crash, bug, erro não tratado).

**Por que acontece:** Step Functions não impõe timeout por padrão — um Task sem `TimeoutSeconds` aguarda até o limite máximo do workflow (1 ano em Standard). O desenvolvedor testa o caminho feliz e nunca percebe o comportamento em falha.

**Como evitar:**
- **Sempre** defina `TimeoutSeconds` em Tasks com `waitForTaskToken`. Use um valor realista para o SLA do processo (ex: 86400 para aprovações de 24h).
- Defina também `HeartbeatSeconds` se o worker deve confirmar progresso periodicamente.
- Configure um `Catch` para `States.Timeout` e `States.HeartbeatTimeout` que transicione para um estado de compensação ou alerta.

```json
"AguardandoAprovacao": {
  "Type": "Task",
  "Resource": "arn:aws:states:::sqs:sendMessage.waitForTaskToken",
  "TimeoutSeconds": 86400,
  "HeartbeatSeconds": 3600,
  "Catch": [
    {
      "ErrorEquals": ["States.Timeout", "States.HeartbeatTimeout"],
      "Next": "AprovacaoExpirada"
    }
  ]
}
```

---

## Exercício de reflexão

Você está projetando um sistema de onboarding de clientes corporativos. O processo envolve: (1) validar os dados do cadastro via API interna, (2) enviar documentos para análise de compliance (que pode levar até 5 dias úteis), (3) aguardar aprovação manual de um analista, (4) provisionar acesso ao sistema quando aprovado, ou (5) notificar o cliente e encerrar se reprovado.

**Questão:** Como você escolheria entre Standard Workflow e Express Workflow para este caso? Quais states usaria em cada etapa (Task, Wait, Choice, Succeed, Fail)? Em particular, como modelaria o passo 3 (aprovação manual) para que o analista possa aprovar ou reprovar via uma API REST, sem que a execução bloqueie um servidor? Descreva também como você leria o histórico de uma execução que falhou no passo 4 para identificar a causa raiz.

---

## Recursos para aprofundar

1. **AWS Step Functions Developer Guide — Welcome**
   URL: https://docs.aws.amazon.com/step-functions/latest/dg/welcome.html
   O ponto de entrada da documentação oficial. A seção "Concepts" cobre a terminologia (state machine, execution, state, transition) com precisão. Leia especialmente "How Step Functions Works" para entender o modelo de execução.

2. **Choosing workflow type — Standard vs Express**
   URL: https://docs.aws.amazon.com/step-functions/latest/dg/concepts-standard-vs-express.html
   A tabela comparativa oficial com todos os parâmetros: guarantees, duração, pricing, logging, throughput e casos de uso. É a referência primária para decisão de tipo de workflow.

3. **Amazon States Language specification**
   URL: https://states-language.net/spec.html
   A especificação completa e aberta da linguagem ASL. Cobre cada tipo de estado, JsonPath no contexto da ASL (incluindo Context Object com `$$`), campos obrigatórios e opcionais, operadores de Choice, e comportamento de error handling. Mais preciso que qualquer tutorial para entender edge cases.

---

