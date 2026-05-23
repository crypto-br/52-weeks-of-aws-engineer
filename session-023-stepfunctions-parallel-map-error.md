# Sessão 23 — Step Functions: Parallel, Map, fluxo de dados entre states e error handling

**Duração estimada:** 60 minutos
**Pré-requisitos:** session-022-stepfunctions-standard-express-states

---

## Objetivo

Ao final, você conseguirá usar Parallel para execução simultânea de branches independentes e Map para iteração sobre arrays (modo inline vs distributed), dominar os campos InputPath, OutputPath, ResultPath e Parameters para controlar o fluxo de dados entre states sem surpresas, e implementar Retry com backoff exponencial e Catch por tipo de erro com redirecionamento a um estado de fallback.

---

## Contexto

[FATO] A sessão anterior cobriu os estados básicos de Step Functions (Task, Choice, Wait, Succeed, Fail) e as diferenças entre Standard e Express Workflows. Esta sessão avança para os três mecanismos que tornam Step Functions útil em cenários reais de produção: paralelismo estruturado (Parallel), iteração sobre coleções (Map) e tratamento declarativo de falhas (Retry + Catch).

[CONSENSO] O fluxo de dados entre estados — controlado pelos campos InputPath, Parameters, ResultSelector, ResultPath e OutputPath — é consistentemente apontado pela comunidade como a parte mais confusa de Step Functions. A maioria dos bugs em workflows reais vem de desenvolvedores que não entendem a ordem de aplicação desses filtros, resultando em dados perdidos ou sobrepostos silenciosamente. Dominar esse pipeline é tão importante quanto conhecer os tipos de estado.

[FATO] O estado `Map` ganhou um segundo modo de operação — **Distributed** — que permite até 10.000 iterações paralelas rodando como execuções filho independentes. Esse modo foi introduzido em 2022 e é a base para pipelines de processamento de dados em larga escala diretamente em Step Functions, sem necessidade de ferramentas externas como AWS Glue para orquestração simples.

---

## Conceitos principais

### 1. Parallel State — branches concorrentes com output agregado

[FATO] O estado `Parallel` executa múltiplos branches (sub-workflows) simultaneamente. Cada branch é uma mini state machine completa com `StartAt` e `States`. Step Functions aguarda que **todos os branches** cheguem a um estado terminal antes de avançar para o `Next` do Parallel.

```
Parallel State
───────────────────────────────────────────────────────────────────────
                    ┌──── Branch 1 ─────┐
Input ──► Parallel ─┤                   ├──► Output (array[branch1, branch2, branch3])
                    ├──── Branch 2 ─────┤
                    └──── Branch 3 ─────┘
                    (todos rodam ao mesmo tempo)
                    (espera o mais lento)
```

[FATO] O output do Parallel é sempre um **array** com um elemento por branch, na ordem em que foram declarados. O output de cada branch é o output do seu último estado.

```json
"EnriquecerPedido": {
  "Type": "Parallel",
  "Branches": [
    {
      "StartAt": "BuscarCliente",
      "States": {
        "BuscarCliente": {
          "Type": "Task",
          "Resource": "arn:aws:states:::lambda:invoke",
          "Parameters": {
            "FunctionName": "BuscarCliente",
            "Payload.$": "$"
          },
          "ResultSelector": { "cliente.$": "$.Payload" },
          "End": true
        }
      }
    },
    {
      "StartAt": "BuscarEstoque",
      "States": {
        "BuscarEstoque": {
          "Type": "Task",
          "Resource": "arn:aws:states:::lambda:invoke",
          "Parameters": {
            "FunctionName": "BuscarEstoque",
            "Payload.$": "$"
          },
          "ResultSelector": { "estoque.$": "$.Payload" },
          "End": true
        }
      }
    }
  ],
  "ResultPath": "$.enriquecimento",
  "Next": "ProcessarPedidoEnriquecido"
}
```

Após o Parallel acima, `$.enriquecimento` será:
```json
[
  { "cliente": { "nome": "...", "credito": 5000 } },
  { "estoque": { "sku": "P001", "disponivel": 12 } }
]
```

[FATO] Se **qualquer branch** falhar com um erro não tratado dentro do branch, o estado Parallel inteiro falha imediatamente (os outros branches são cancelados). O Parallel pode ter seus próprios campos `Retry` e `Catch` para tratar falhas de qualquer branch.

[CONSENSO] Uma armadilha comum é esperar que cada branch receba um input diferente. Na verdade, todos os branches recebem o **mesmo input** — o input efetivo do estado Parallel (após InputPath/Parameters). Para passar dados diferentes a cada branch, transforme o input dentro do primeiro estado de cada branch.

---

### 2. Map State — iteração sobre arrays (Inline vs Distributed)

[FATO] O estado `Map` itera sobre um array (no input ou em fonte externa) e executa um sub-workflow para cada item. Os dois modos têm características radicalmente diferentes:

```
┌──────────────────┬───────────────────────────┬────────────────────────────────┐
│ Dimensão         │ Inline Map                │ Distributed Map                │
├──────────────────┼───────────────────────────┼────────────────────────────────┤
│ MaxConcurrency   │ Até 40                    │ Até 10.000                     │
├──────────────────┼───────────────────────────┼────────────────────────────────┤
│ Fonte dos itens  │ Array no input JSON       │ Array JSON, CSV ou lista S3    │
├──────────────────┼───────────────────────────┼────────────────────────────────┤
│ Execução filho   │ Inline (mesmo execution)  │ Child workflow (execução própria│
├──────────────────┼───────────────────────────┼────────────────────────────────┤
│ Execution history│ Embutido no pai           │ Separado por iteração          │
├──────────────────┼───────────────────────────┼────────────────────────────────┤
│ Resultados       │ Array no output do Map    │ Pode escrever em S3            │
├──────────────────┼───────────────────────────┼────────────────────────────────┤
│ Tolerância falha │ ToleratedFailureCount/     │ ToleratedFailureCount/         │
│                  │ Percentage                │ Percentage                     │
└──────────────────┴───────────────────────────┴────────────────────────────────┘
```

#### Map Inline

```json
"ProcessarItens": {
  "Type": "Map",
  "ItemsPath": "$.itens",
  "ItemSelector": {
    "item_id.$": "$$.Map.Item.Value.id",
    "indice.$": "$$.Map.Item.Index",
    "contexto.$": "$.contexto_global"
  },
  "MaxConcurrency": 10,
  "ToleratedFailurePercentage": 20,
  "Iterator": {
    "StartAt": "ProcessarUmItem",
    "States": {
      "ProcessarUmItem": {
        "Type": "Task",
        "Resource": "arn:aws:states:::lambda:invoke",
        "Parameters": {
          "FunctionName": "ProcessarItem",
          "Payload.$": "$"
        },
        "ResultSelector": { "resultado.$": "$.Payload" },
        "End": true
      }
    }
  },
  "ResultPath": "$.resultados",
  "Next": "Finalizar"
}
```

[FATO] Campos importantes do Map:

| Campo | Descrição |
|---|---|
| `ItemsPath` | JsonPath para o array no input. Default: `$` (input inteiro deve ser array) |
| `ItemSelector` | Constrói o input de cada iteração. `$$.Map.Item.Value` = item atual; `$$.Map.Item.Index` = índice |
| `MaxConcurrency` | Máximo de iterações paralelas. `0` = sem limite (até 40 em Inline) |
| `ToleratedFailureCount` | Número absoluto de falhas toleradas antes do Map falhar |
| `ToleratedFailurePercentage` | Percentual de falhas toleradas (0-100) |

[FATO] `$$.Map.Item.Value` e `$$.Map.Item.Index` são referências ao **Context Object** específicas do Map — acessíveis apenas dentro de `ItemSelector`. `$$` acessa o contexto de execução; `$` acessa o item já transformado pelo ItemSelector.

#### Map Distributed

```json
"ProcessarCSVGrande": {
  "Type": "Map",
  "ItemReader": {
    "Resource": "arn:aws:states:::s3:getObject",
    "ReaderConfig": {
      "InputType": "CSV",
      "CSVHeaderLocation": "FIRST_ROW"
    },
    "Parameters": {
      "Bucket": "meu-bucket",
      "Key": "dados/pedidos.csv"
    }
  },
  "MaxConcurrency": 1000,
  "ToleratedFailurePercentage": 5,
  "ItemSelector": {
    "linha.$": "$$.Map.Item.Value"
  },
  "ItemBatcher": {
    "MaxItemsPerBatch": 10,
    "MaxInputBytesPerBatch": 262144
  },
  "Iterator": {
    "StartAt": "ProcessarLote",
    "States": {
      "ProcessarLote": {
        "Type": "Task",
        "Resource": "arn:aws:states:::lambda:invoke",
        "Parameters": {
          "FunctionName": "ProcessarLote",
          "Payload.$": "$"
        },
        "End": true
      }
    }
  },
  "ResultWriter": {
    "Resource": "arn:aws:states:::s3:putObject",
    "Parameters": {
      "Bucket": "meu-bucket",
      "Prefix": "resultados/"
    }
  },
  "Next": "Finalizar"
}
```

[FATO] `ItemBatcher` é exclusivo do modo Distributed: agrupa múltiplos itens do array em um único input para cada iteração, reduzindo o número de invocações e aumentando eficiência para cargas com muitos itens pequenos.

[FATO] `ResultWriter` escreve os resultados de todas as iterações em S3 em vez de retorná-los inline no output do Map — essencial quando o volume de resultados excederia o limite de 256KB do payload de estado.

---

### 3. Pipeline de dados entre estados — a ordem importa

[FATO] Step Functions aplica cinco filtros em sequência antes de passar o controle para o próximo estado. Entender essa ordem é crítico para não perder dados ou corromper o fluxo:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PIPELINE DE DADOS EM UM ESTADO

  Raw Input (output do estado anterior)
       │
       ▼
  ┌──────────┐
  │InputPath │  Filtra o input bruto. Default: "$" (tudo).
  │          │  null → passa {} vazio para Parameters.
  └────┬─────┘
       │ Effective Input
       ▼
  ┌────────────┐
  │ Parameters │  Constrói o payload enviado ao serviço.
  │            │  Chaves com ".$" são referências JsonPath no Effective Input.
  └─────┬──────┘
        │ Task Input
        ▼
  ╔═════════╗
  ║  SERVIÇO ║  Lambda / DynamoDB / SQS / etc.
  ╚═════╤═══╝
        │ Raw Result (response do serviço)
        ▼
  ┌────────────────┐
  │ ResultSelector │  Filtra/reshapes o resultado bruto.
  │                │  Chaves com ".$" são referências no Raw Result.
  └───────┬────────┘
          │ Selected Result
          ▼
  ┌────────────┐
  │ ResultPath │  Onde gravar o Selected Result no Effective Input.
  │            │  "$"  → substitui o Effective Input inteiro.
  │            │  null → descarta o resultado, passa Effective Input.
  │            │  "$.x"→ adiciona/sobrescreve o campo "x".
  └──────┬─────┘
         │ Effective Output
         ▼
  ┌────────────┐
  │ OutputPath │  Filtra o Effective Output antes de enviar.
  │            │  Default: "$" (tudo). null → passa {}.
  └──────┬─────┘
         │
         ▼
  Input do próximo estado
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

[FATO] Exemplo concreto para fixar cada passo:

```json
// Input bruto recebido pelo estado:
// { "pedido": { "id": "P001", "valor": 1500 }, "contexto": { "usuario": "U42" } }

{
  "Type": "Task",
  "Resource": "arn:aws:states:::lambda:invoke",

  // 1. InputPath: usa só o campo "pedido" como effective input
  "InputPath": "$.pedido",
  // effective input: { "id": "P001", "valor": 1500 }

  // 2. Parameters: constrói o payload para a Lambda
  "Parameters": {
    "FunctionName": "ValidarPedido",
    "Payload": {
      "pedido_id.$": "$.id",
      "valor.$": "$.valor",
      "timestamp.$": "$$.Execution.StartTime"
    }
  },
  // task input para Lambda: { "pedido_id": "P001", "valor": 1500, "timestamp": "2026-06-22T..." }

  // 3. [Lambda executa e retorna]:
  // { "StatusCode": 200, "Payload": { "aprovado": true, "score": 95 } }

  // 4. ResultSelector: pega só o que interessa do resultado da Lambda
  "ResultSelector": {
    "aprovado.$": "$.Payload.aprovado",
    "score.$":    "$.Payload.score"
  },
  // selected result: { "aprovado": true, "score": 95 }

  // 5. ResultPath: grava no effective input ORIGINAL (InputPath=$.pedido)
  //    ATENÇÃO: ResultPath é aplicado sobre o effective input (após InputPath),
  //    não sobre o raw input.
  "ResultPath": "$.validacao",
  // effective output: { "id": "P001", "valor": 1500, "validacao": { "aprovado": true, "score": 95 } }

  // 6. OutputPath: envia apenas o necessário
  "OutputPath": "$.validacao",
  // output para próximo estado: { "aprovado": true, "score": 95 }

  "Next": "VerificarAprovacao"
}
```

[FATO] Ponto crítico frequentemente confundido: `ResultPath` é aplicado sobre o **effective input** (saída do InputPath), não sobre o raw input original. Se `InputPath` filtrou para `$.pedido`, então o effective input já não tem `$.contexto`. Ao usar `ResultPath: "$.validacao"`, o resultado é adicionado ao `$.pedido` filtrado, não ao input original completo. Para preservar o input completo, evite usar InputPath ou use `InputPath: "$"`.

---

### 4. Error Handling — Retry e Catch

[FATO] Error handling em Step Functions é declarativo: você especifica, dentro do estado, como reagir a erros — sem código externo ou lambdas de retry. Retry é sempre avaliado antes de Catch.

#### Códigos de erro built-in

[FATO] Step Functions define um conjunto de erros reservados com prefixo `States.`:

```
States.ALL                    — wildcard: casa com qualquer erro
States.TaskFailed             — a Task retornou falha
States.Timeout                — TimeoutSeconds excedido
States.HeartbeatTimeout       — HeartbeatSeconds excedido sem heartbeat
States.NoChoiceMatched        — Choice sem Default e nenhuma regra casou
States.ResultPathMatchFailure — ResultPath não pôde ser aplicado
States.ParameterPathFailure   — referência JsonPath falhou em Parameters
States.BranchFailed           — branch de Parallel falhou
States.ExceedToleratedFailureThreshold — Map ultrapassou tolerância de falha
States.Runtime                — erro de runtime não classificado
```

Erros de serviços externos seguem a convenção `<Serviço>.<TipoErro>`, por exemplo:
- `Lambda.ServiceException` — erro interno da Lambda
- `Lambda.TooManyRequestsException` — throttling
- `Lambda.AWSLambdaException` — erro de execução da função
- `DynamoDB.ProvisionedThroughputExceededException`

#### Retry — backoff exponencial com jitter

[FATO] O array `Retry` é avaliado na ordem. O **primeiro** `ErrorEquals` que casar é aplicado. Erros específicos devem vir antes de `States.ALL`.

```json
"ProcessarPagamento": {
  "Type": "Task",
  "Resource": "arn:aws:states:::lambda:invoke",
  "Parameters": { "FunctionName": "ProcessarPagamento", "Payload.$": "$" },
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
      "ErrorEquals": ["PagamentoTemporarioIndisponivel"],
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
  "Next": "PagamentoAprovado"
}
```

[FATO] Campos de Retry:

| Campo | Obrigatório | Default | Descrição |
|---|---|---|---|
| `ErrorEquals` | Sim | — | Array de códigos de erro que ativam esta regra |
| `IntervalSeconds` | Não | 1 | Espera inicial antes do primeiro retry |
| `MaxAttempts` | Não | 3 | Total de tentativas (0 = sem retry) |
| `BackoffRate` | Não | 2.0 | Multiplicador aplicado a cada retry |
| `MaxDelaySeconds` | Não | sem limite | Cap no intervalo máximo entre retries |
| `JitterStrategy` | Não | `NONE` | `FULL` adiciona jitter aleatório (0 a intervalo calculado) |

[FATO] Cálculo do intervalo com backoff e jitter `FULL`:
```
Tentativa 1: random(0, 1)   segundos    (IntervalSeconds=1, BackoffRate=2)
Tentativa 2: random(0, 2)   segundos
Tentativa 3: random(0, 4)   segundos
Tentativa 4: random(0, 8)   segundos  → se MaxDelaySeconds=10, cap em 10
Tentativa 5: random(0, 10)  segundos
```

[CONSENSO] `JitterStrategy: FULL` é recomendado quando múltiplas execuções paralelas podem falhar simultaneamente (ex: múltiplas Lambdas atingindo o mesmo serviço throttled). Sem jitter, todas retrys acontecem nos mesmos intervalos, criando ondas de carga — o fenômeno chamado *thundering herd*.

#### Catch — fallback após retries esgotados

[FATO] Após todos os retries de uma regra falharem, `Catch` é avaliado. O array de `Catch` também é avaliado em ordem — primeiro match ganha.

```json
"Catch": [
  {
    "ErrorEquals": ["PedidoDuplicado", "LimiteCredito"],
    "ResultPath": "$.erro_negocio",
    "Next": "TratarErroDenegocio"
  },
  {
    "ErrorEquals": ["States.Timeout"],
    "ResultPath": "$.erro_timeout",
    "Next": "NotificarTimeout"
  },
  {
    "ErrorEquals": ["States.ALL"],
    "ResultPath": "$.erro",
    "Next": "FallbackGenerico"
  }
]
```

[FATO] Quando o Catch é acionado, Step Functions injeta no estado seguinte um objeto de erro com os campos `Error` e `Cause`. O `ResultPath` controla onde esse objeto é inserido:

```json
// Sem ResultPath no Catch (ou ResultPath: "$"):
// Input do estado de fallback = { "Error": "Lambda.ServiceException", "Cause": "..." }
// (o input original é completamente substituído)

// Com ResultPath: "$.erro":
// Input do estado de fallback = { ...input_original..., "erro": { "Error": "...", "Cause": "..." } }
// (input original preservado + erro adicionado)
```

[CONSENSO] **Sempre** use `ResultPath` em Catch para preservar o contexto original. Sem ele, o estado de fallback recebe apenas a mensagem de erro, sem os dados do pedido/cliente/etc. que seriam necessários para logar, compensar ou notificar.

#### Fluxo completo: Retry → Catch

```
Estado Task é invocado
       │
       ▼ erro ocorre
┌──────────────┐
│ Retry[0]     │ → Verifica ErrorEquals → Match?
│ ErrorEquals  │      Sim → aguarda IntervalSeconds → reinvoca → erro → Retry[1]?
│ MaxAttempts  │                                              → sucesso → Next
└──────┬───────┘
       │ MaxAttempts esgotados
       ▼
┌──────────────┐
│ Retry[1]     │ → próximo retry rule
└──────┬───────┘
       │ todos os retries esgotados
       ▼
┌──────────────┐
│ Catch[0]     │ → Verifica ErrorEquals → Match → Next (fallback state)
│ Catch[1]     │
└──────┬───────┘
       │ nenhum Catch casou
       ▼
  Execução FAILED
```

---

## Exemplo prático

**Cenário:** Pipeline de processamento de notas fiscais. Recebe um array de NFes, valida cada uma em paralelo com enriquecimento fiscal e de cliente, e processa o resultado.

```json
{
  "Comment": "Pipeline de processamento de notas fiscais",
  "StartAt": "ProcessarNFes",
  "States": {

    "ProcessarNFes": {
      "Type": "Map",
      "ItemsPath": "$.notas",
      "ItemSelector": {
        "nota.$": "$$.Map.Item.Value",
        "indice.$": "$$.Map.Item.Index"
      },
      "MaxConcurrency": 20,
      "ToleratedFailurePercentage": 10,
      "Iterator": {
        "StartAt": "EnriquecerNF",
        "States": {

          "EnriquecerNF": {
            "Type": "Parallel",
            "Branches": [
              {
                "StartAt": "ValidarFisco",
                "States": {
                  "ValidarFisco": {
                    "Type": "Task",
                    "Resource": "arn:aws:states:::lambda:invoke",
                    "Parameters": {
                      "FunctionName": "ValidarFisco",
                      "Payload.$": "$.nota"
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
                "StartAt": "BuscarDadosCliente",
                "States": {
                  "BuscarDadosCliente": {
                    "Type": "Task",
                    "Resource": "arn:aws:states:::dynamodb:getItem.sync:2",
                    "Parameters": {
                      "TableName": "clientes",
                      "Key": {
                        "cnpj": { "S.$": "$.nota.emitente_cnpj" }
                      }
                    },
                    "ResultSelector": { "cliente.$": "$.Item" },
                    "Catch": [
                      {
                        "ErrorEquals": ["DynamoDB.ResourceNotFoundException"],
                        "ResultPath": "$.erro_cliente",
                        "Next": "ClienteNaoEncontrado"
                      }
                    ],
                    "End": true
                  },
                  "ClienteNaoEncontrado": {
                    "Type": "Pass",
                    "Result": { "cliente": null },
                    "End": true
                  }
                }
              }
            ],
            "ResultSelector": {
              "fiscal.$":  "$[0].fiscal",
              "cliente.$": "$[1].cliente"
            },
            "ResultPath": "$.enriquecimento",
            "Next": "VerificarFraude"
          },

          "VerificarFraude": {
            "Type": "Choice",
            "Choices": [
              {
                "Variable": "$.enriquecimento.fiscal.situacao",
                "StringEquals": "IRREGULAR",
                "Next": "BloquearNF"
              }
            ],
            "Default": "RegistrarNF"
          },

          "RegistrarNF": {
            "Type": "Task",
            "Resource": "arn:aws:states:::dynamodb:putItem.sync:2",
            "Parameters": {
              "TableName": "notas_processadas",
              "Item": {
                "id":       { "S.$": "$.nota.chave_acesso" },
                "fiscal":   { "S.$": "States.JsonToString($.enriquecimento.fiscal)" },
                "status":   { "S": "PROCESSADA" }
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
                "ResultPath": "$.erro_registro",
                "Next": "FalhaRegistro"
              }
            ],
            "End": true
          },

          "BloquearNF": {
            "Type": "Fail",
            "Error": "NFIrregular",
            "Cause": "Nota fiscal com situação fiscal irregular"
          },

          "FalhaRegistro": {
            "Type": "Fail",
            "Error": "FalhaRegistro",
            "CausePath": "$.erro_registro.Cause"
          }
        }
      },
      "ResultPath": "$.resultados",
      "Next": "GerarRelatorio"
    },

    "GerarRelatorio": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "GerarRelatorio",
        "Payload": {
          "total.$": "States.ArrayLength($.resultados)",
          "resultados.$": "$.resultados"
        }
      },
      "ResultPath": null,
      "End": true
    }

  }
}
```

### CDK — Parallel, Map e error handling em Python

```python
from aws_cdk import (
    Stack, Duration,
    aws_stepfunctions as sfn,
    aws_stepfunctions_tasks as tasks,
    aws_lambda as lambda_,
    aws_dynamodb as dynamodb,
)

class NFWorkflowStack(Stack):

    def __init__(self, scope, construct_id, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        tabela = dynamodb.Table.from_table_name(self, "Tabela", "notas_processadas")
        fn_validar = lambda_.Function.from_function_name(self, "FnValidar", "ValidarFisco")
        fn_relatorio = lambda_.Function.from_function_name(self, "FnRelatorio", "GerarRelatorio")

        # ── Branches do Parallel ──────────────────────────────────────
        validar_fisco = tasks.LambdaInvoke(
            self, "ValidarFisco",
            lambda_function=fn_validar,
            payload=sfn.TaskInput.from_json_path_at("$.nota"),
            result_selector={"fiscal": sfn.JsonPath.object_at("$.Payload")},
        ).add_retry(
            errors=["Lambda.TooManyRequestsException"],
            interval=Duration.seconds(2),
            max_attempts=3,
            backoff_rate=2,
            jitter_strategy=sfn.JitterType.FULL,
        )

        cliente_nao_encontrado = sfn.Pass(
            self, "ClienteNaoEncontrado",
            result=sfn.Result.from_object({"cliente": None}),
        )

        buscar_cliente = tasks.DynamoGetItem(
            self, "BuscarCliente",
            table=tabela,
            key={"cnpj": tasks.DynamoAttributeValue.from_string(
                sfn.JsonPath.string_at("$.nota.emitente_cnpj")
            )},
            result_selector={"cliente": sfn.JsonPath.object_at("$.Item")},
        ).add_catch(
            cliente_nao_encontrado,
            errors=["DynamoDB.ResourceNotFoundException"],
            result_path="$.erro_cliente",
        )

        enriquecer = sfn.Parallel(
            self, "EnriquecerNF",
            result_selector={
                "fiscal":  sfn.JsonPath.object_at("$[0].fiscal"),
                "cliente": sfn.JsonPath.object_at("$[1].cliente"),
            },
            result_path="$.enriquecimento",
        )
        enriquecer.branch(validar_fisco)
        enriquecer.branch(buscar_cliente)

        # ── Fallbacks e estados terminais ────────────────────────────
        bloquear_nf = sfn.Fail(
            self, "BloquearNF",
            error="NFIrregular",
            cause="Nota fiscal com situação fiscal irregular",
        )

        falha_registro = sfn.Fail(
            self, "FalhaRegistro",
            error="FalhaRegistro",
        )

        # ── Registrar NF ──────────────────────────────────────────────
        registrar_nf = tasks.DynamoPutItem(
            self, "RegistrarNF",
            table=tabela,
            item={
                "id":     tasks.DynamoAttributeValue.from_string(
                              sfn.JsonPath.string_at("$.nota.chave_acesso")),
                "status": tasks.DynamoAttributeValue.from_string("PROCESSADA"),
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
            falha_registro,
            errors=["States.ALL"],
            result_path="$.erro_registro",
        )

        # ── Choice ────────────────────────────────────────────────────
        verificar_fraude = sfn.Choice(self, "VerificarFraude") \
            .when(
                sfn.Condition.string_equals("$.enriquecimento.fiscal.situacao", "IRREGULAR"),
                bloquear_nf,
            ) \
            .otherwise(registrar_nf)

        # ── Iterator do Map ───────────────────────────────────────────
        iterator = enriquecer.next(verificar_fraude)

        # ── Map principal ─────────────────────────────────────────────
        processar_nfes = sfn.Map(
            self, "ProcessarNFes",
            items_path=sfn.JsonPath.string_at("$.notas"),
            item_selector={
                "nota":   sfn.JsonPath.object_at("$$.Map.Item.Value"),
                "indice": sfn.JsonPath.number_at("$$.Map.Item.Index"),
            },
            max_concurrency=20,
            tolerated_failure_percentage=10,
            result_path="$.resultados",
        ).iterator(iterator)

        gerar_relatorio = tasks.LambdaInvoke(
            self, "GerarRelatorio",
            lambda_function=fn_relatorio,
            result_path=sfn.JsonPath.DISCARD,
        )

        definition = processar_nfes.next(gerar_relatorio)

        sfn.StateMachine(
            self, "NFWorkflow",
            state_machine_name="ProcessamentoNF",
            definition_body=sfn.DefinitionBody.from_chainable(definition),
            state_machine_type=sfn.StateMachineType.STANDARD,
        )
```

---

## Armadilhas comuns

### Armadilha 1 — `ResultPath` no Catch sem preservar o input original causa perda de contexto

**O erro:** O desenvolvedor configura um Catch sem `ResultPath` (ou com `ResultPath: "$"`). No estado de fallback, o input inteiro é o objeto `{ "Error": "...", "Cause": "..." }` — o pedido, o cliente, o contexto de execução: tudo desapareceu.

**Por que acontece:** O comportamento padrão de `ResultPath` ausente em Catch é `"$"`, que substitui o effective input inteiro pelo objeto de erro. É o mesmo comportamento do Task, só que aqui o "resultado" é o erro.

**Como evitar:** Sempre use `"ResultPath": "$.erro"` (ou qualquer campo reservado) no Catch. O estado de fallback receberá o input original intacto mais o campo `$.erro` com `{ "Error": "...", "Cause": "..." }`.

```json
// Errado — perde contexto
"Catch": [{ "ErrorEquals": ["States.ALL"], "Next": "Fallback" }]

// Correto — preserva contexto
"Catch": [{ "ErrorEquals": ["States.ALL"], "ResultPath": "$.erro", "Next": "Fallback" }]
```

---

### Armadilha 2 — Usar `InputPath` quando queria `Parameters`, perdendo campos do input

**O erro:** O desenvolvedor quer passar apenas `$.pedido_id` para a Lambda. Ele usa `"InputPath": "$.pedido_id"`. O resultado: a Lambda recebe apenas a string do ID (ex: `"P001"`), não um objeto. Além disso, o effective input para o resto do pipeline (ResultPath, etc.) também é agora apenas essa string.

**Por que acontece:** `InputPath` filtra o **effective input inteiro** — se o valor selecionado é uma string, o effective input passa a ser uma string. `Parameters` deveria ser usado para construir um payload estruturado sem perder o input.

**Como evitar:**
- Use `InputPath` apenas quando quer **selecionar uma sub-árvore** do input que será o novo effective input (mantendo o objeto).
- Use `Parameters` quando quer **construir um novo objeto** com campos selecionados do input.

```json
// Errado: effective input vira string "P001"
"InputPath": "$.pedido_id"

// Correto: effective input vira { "id": "P001", "flag": true }
"Parameters": {
  "id.$": "$.pedido_id",
  "flag": true
}
```

---

### Armadilha 3 — Colocar `States.ALL` antes de erros específicos no Retry ou Catch

**O erro:** O array de Retry ou Catch começa com `{ "ErrorEquals": ["States.ALL"], ... }`. Isso faz com que **todos** os erros — incluindo os que tinham tratamento específico planejado — caiam na regra genérica. As regras subsequentes nunca são avaliadas.

**Por que acontece:** Tanto Retry quanto Catch avaliam as regras em **ordem** e param no primeiro match. `States.ALL` casa com qualquer erro, então bloqueia tudo que vem depois.

**Como evitar:** Sempre coloque erros específicos primeiro, `States.ALL` por último:

```json
// Errado
"Retry": [
  { "ErrorEquals": ["States.ALL"], "MaxAttempts": 3 },
  { "ErrorEquals": ["Lambda.TooManyRequestsException"], "MaxAttempts": 10 }
]

// Correto
"Retry": [
  { "ErrorEquals": ["Lambda.TooManyRequestsException"], "MaxAttempts": 10, "BackoffRate": 2 },
  { "ErrorEquals": ["Lambda.ServiceException"], "MaxAttempts": 5, "BackoffRate": 1.5 },
  { "ErrorEquals": ["States.ALL"], "MaxAttempts": 2, "BackoffRate": 1 }
]
```

---

## Exercício de reflexão

Você está modelando um workflow de importação de dados de um arquivo CSV armazenado no S3. O arquivo tem ~50.000 linhas. Para cada linha, sua equipe precisa: (1) enriquecer com dados de uma API externa (com SLA de 200ms e rate limit de 1.000 req/s), (2) validar regras de negócio, e (3) persistir em DynamoDB.

**Questão:** Como você escolheria entre Map Inline e Map Distributed para este caso? Qual seria o `MaxConcurrency` adequado para respeitar o rate limit da API externa? Onde colocaria o `ResultWriter` e por quê? Descreva também como configuraria o Retry para o passo 1 (chamada à API externa com rate limit) e o Catch para o passo 3 (falha no DynamoDB), de forma que linhas com erro de persistência sejam marcadas para reprocessamento sem cancelar o processamento das demais.

---

## Recursos para aprofundar

1. **AWS Step Functions — Map State**
   URL: https://docs.aws.amazon.com/step-functions/latest/dg/state-map.html
   Ponto de entrada para a documentação do Map state, com links para os dois modos (Inline e Distributed). A seção "Inline Mode" detalha ItemsPath, ItemSelector e MaxConcurrency; a seção "Distributed Mode" cobre ItemReader, ItemBatcher e ResultWriter com exemplos de CSV e S3.

2. **Processing input and output in Step Functions**
   URL: https://docs.aws.amazon.com/step-functions/latest/dg/concepts-input-output-filtering.html
   Referência completa do pipeline InputPath → Parameters → ResultSelector → ResultPath → OutputPath. Inclui exemplos visuais do estado dos dados em cada etapa, com tabelas mostrando o antes/depois de cada filtro.

3. **Handling errors in Step Functions workflows**
   URL: https://docs.aws.amazon.com/step-functions/latest/dg/concepts-error-handling.html
   Lista completa de códigos de erro `States.*`, semântica de Retry (campos, BackoffRate, JitterStrategy, MaxDelaySeconds) e Catch (ordem de avaliação, interação com ResultPath). Inclui exemplos de estados com ambos configurados.

---

