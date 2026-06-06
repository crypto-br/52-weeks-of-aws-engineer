# Sessão 037 — CloudWatch Logs Insights: sintaxe de queries e campos derivados

**Duração estimada:** 60 minutos  
**Dependências:** session-036-cloudwatch-custom-metrics-emf

---

## Objetivo

Ao final desta sessão, você conseguirá escrever queries Logs Insights para agregar erros por endpoint, extrair campos de logs JSON estruturados com `parse`, criar visualizações de timeseries, usar campos descobertos automaticamente (`@timestamp`, `@message`, `@requestId`), e salvar queries reutilizáveis no console.

---

## Contexto

[FATO] CloudWatch Logs Insights é um serviço de análise interativa de logs que aceita uma linguagem de query própria orientada a pipes. Você pode consultar múltiplos log groups simultaneamente, visualizar resultados como timeseries ou tabelas, e salvar queries para reutilização. Os resultados são gerados em segundos a minutos dependendo do volume.

[FATO] O CloudWatch Logs Insights tem custo por GB de dados escaneados (us-east-1: $0.005/GB). Quanto mais preciso o filtro temporal e por log group, menor o custo. Queries não têm custo de request — apenas de scanning.

---

## Conceitos principais

### 1. Campos descobertos automaticamente

[FATO] Logs Insights automaticamente descobre e indexa certos campos:

```
Campo           Origem / Conteúdo
────────────────────────────────────────────────────────────────
@timestamp      Timestamp do log event (sempre disponível)
@message        Texto completo do log event
@logStream      Nome do log stream
@log            Identificador do log group (account-id:log-group-name)
@requestId      Presente em logs Lambda REPORT e START/END
@duration       Duração de invocação Lambda (em ms) nos REPORT events
@billedDuration Duração faturável Lambda (ms)
@memorySize     Memória configurada (bytes)
@maxMemoryUsed  Memória máxima usada (bytes)
@initDuration   Duração do cold start (ms) — apenas em cold starts
@type           Tipo de event Lambda: "START", "END", "REPORT"
```

[FATO] Para logs em **formato JSON**, Logs Insights descobre automaticamente os campos de primeiro nível como campos pesquisáveis. Um log `{"level": "ERROR", "endpoint": "/pay", "latency": 250}` tem `level`, `endpoint` e `latency` disponíveis diretamente nas queries sem precisar de `parse`.

---

### 2. Comandos principais da linguagem

[FATO] A linguagem é baseada em pipes: cada comando recebe os resultados do anterior. Os comandos são case-insensitive mas os nomes de campos são case-sensitive.

```
Comando         Propósito
────────────────────────────────────────────────────────────────
fields          Seleciona campos a exibir; cria campos derivados
filter          Filtra eventos (WHERE equivalente)
stats           Agrega dados — count, avg, sum, min, max, percentile
sort            Ordena resultados (deve vir após o último stats)
limit           Limita número de linhas retornadas
parse           Extrai subcampos de um campo de texto (glob, regex, logfmt)
dedup           Remove duplicatas por campo(s)
display         Formata a exibição sem afetar filtros (alias de fields pós-stats)
pattern         Agrupa mensagens similares (ML-based clustering)
diff            Compara métricas com período anterior
```

---

### 3. `fields` — projeção e campos derivados

[FATO] O comando `fields` seleciona quais campos exibir e pode criar novos campos com expressões:

```sql
-- Seleção simples
fields @timestamp, @message, level, endpoint

-- Campos derivados com operadores aritméticos
fields @timestamp, @duration / 1000 as durationSeconds

-- Campos derivados condicionais com if()
fields @timestamp,
       if(statusCode >= 400, "error", "ok") as requestStatus

-- coalesce: primeiro não-nulo
fields @timestamp,
       coalesce(httpMethod, method, "UNKNOWN") as verb
```

---

### 4. `filter` — filtragem de eventos

[FATO] Suporta operadores de comparação (`=`, `!=`, `<`, `<=`, `>`, `>=`), `like` (substring), `not like`, `in [...]`, `ispresent()`, `not ispresent()`:

```sql
-- Filtro simples
filter level = "ERROR"

-- Like (substring, case-sensitive)
filter @message like "TimeoutException"

-- Regex com ~
filter @message =~ /TimeoutException|ConnectionReset/

-- Múltiplas condições
filter statusCode >= 400 and endpoint like "/api/payments"

-- Verificar existência de campo
filter ispresent(errorCode)

-- IN para lista de valores
filter statusCode in [400, 401, 403, 404]

-- Combinando com NOT
filter not (statusCode = 200 or statusCode = 201)
```

---

### 5. `stats` — agregações

[FATO] Funções disponíveis em `stats`:

```
Função              Descrição
────────────────────────────────────────────────────────────────
count(*)            Total de eventos no grupo
count(field)        Total de eventos onde field não é nulo
count_distinct(f)   Contagem de valores únicos
sum(field)          Soma
avg(field)          Média aritmética
min(field)          Mínimo
max(field)          Máximo
pct(field, n)       Percentil (ex: pct(@duration, 95) = p95)
stddev(field)       Desvio padrão
earliest(field)     Valor mais antigo no grupo
latest(field)       Valor mais recente no grupo
```

[FATO] `bin(period)` agrupa por janela temporal — fundamental para timeseries:

```sql
-- Erros por endpoint por minuto
filter level = "ERROR"
| stats count(*) as errorCount by endpoint, bin(1m)
| sort bin desc

-- Períodos válidos: 1s, 10s, 30s, 1m, 5m, 10m, 15m, 30m, 1h, 6h, 1d
```

---

### 6. `parse` — extração de campos de texto não estruturado

[FATO] Três modos de `parse`:

**Glob (wildcard `*`):**
```sql
-- Log: "user=alice, method:GET, latency := 45"
parse @message "user=*, method:*, latency := *"
    as user, method, latency
| stats avg(latency) by method
```

**Regex com grupos nomeados:**
```sql
-- Log: "[ERROR] POST /api/pay - 503 - 1250ms"
parse @message /\[(?<logLevel>[A-Z]+)\] (?<httpMethod>[A-Z]+) (?<path>[^ ]+) - (?<statusCode>\d+) - (?<durationMs>\d+)ms/
| filter logLevel = "ERROR"
| stats count(*) by path, statusCode
```

**Para logs JSON estruturados — parse NÃO é necessário:**
```sql
-- Log JSON: {"level":"ERROR","endpoint":"/pay","latencyMs":1250,"userId":"u-123"}
-- Logs Insights já descobre os campos automaticamente

fields @timestamp, endpoint, latencyMs, level
| filter level = "ERROR"
| stats avg(latencyMs) as avgLatency by endpoint
```

---

## Exemplo prático

### Cenário: análise de logs de API de pagamentos

Os logs são emitidos em JSON estruturado pelo Lambda Powertools Logger:

```json
{
  "@timestamp": "2024-01-15T10:23:45.123Z",
  "level": "INFO",
  "service": "payment-processor",
  "message": "Payment processed",
  "endpoint": "/api/v1/payments",
  "httpMethod": "POST",
  "statusCode": 200,
  "latencyMs": 45,
  "userId": "u-abc123",
  "orderId": "ord-xyz789",
  "cold_start": false,
  "xray_trace_id": "1-abc123-def456"
}
```

#### Query 1: Taxa de erros por endpoint nos últimos 60 minutos

```sql
fields @timestamp, endpoint, statusCode, level
| filter level in ["ERROR", "CRITICAL"] or statusCode >= 400
| stats
    count(*) as totalErrors,
    count_distinct(userId) as uniqueUsersAffected,
    max(latencyMs) as maxLatency
  by endpoint, statusCode
| sort totalErrors desc
| limit 20
```

#### Query 2: Percentis de latência por endpoint (timeseries, 5 min)

```sql
filter ispresent(latencyMs) and ispresent(endpoint)
| stats
    avg(latencyMs) as avgLatency,
    pct(latencyMs, 50) as p50,
    pct(latencyMs, 95) as p95,
    pct(latencyMs, 99) as p99,
    count(*) as requestCount
  by endpoint, bin(5m)
| sort bin desc
```

#### Query 3: Cold starts Lambda — identificar e medir impacto

```sql
-- Usa campos especiais do Lambda REPORT
filter @type = "REPORT"
| stats
    count(*) as totalInvocations,
    sum(if(ispresent(@initDuration), 1, 0)) as coldStartCount,
    avg(@initDuration) as avgColdStartMs,
    max(@initDuration) as maxColdStartMs,
    avg(@duration) as avgDurationMs,
    avg(@maxMemoryUsed / 1000 / 1000) as avgMemoryMB,
    max(@memorySize / 1000 / 1000) - max(@maxMemoryUsed / 1000 / 1000)
        as overProvisionedMB
  by bin(1h)
| sort bin desc
```

#### Query 4: Rastrear jornada de um usuário por orderId

```sql
fields @timestamp, level, message, endpoint, statusCode, latencyMs, orderId
| filter orderId = "ord-xyz789"
| sort @timestamp asc
```

#### Query 5: Top 10 erros agrupados por mensagem

```sql
filter level = "ERROR"
| stats count(*) as occurrences by message
| sort occurrences desc
| limit 10
```

#### Query 6: `parse` em logs não-estruturados — latência de REPORT Lambda

```sql
-- Lambda REPORT logs: "REPORT RequestId: abc Duration: 123.45 ms Billed Duration: 200 ms ..."
filter @type = "REPORT"
| parse @message "Duration: * ms" as rawDuration
| stats
    avg(rawDuration) as avgDuration,
    pct(rawDuration, 99) as p99Duration,
    max(rawDuration) as maxDuration
  by bin(5m)
| sort bin desc
```

#### Query 7: `dedup` — última ocorrência de erro por usuário

```sql
fields @timestamp, userId, message, endpoint, statusCode
| filter level = "ERROR" and ispresent(userId)
| sort @timestamp desc
| dedup userId
| limit 50
```

#### Query 8: Requests lentos com deduplicação de retries

```sql
fields @timestamp, @requestId, endpoint, latencyMs, statusCode, userId
| filter latencyMs > 1000
| sort @timestamp desc
| dedup @requestId
| limit 20
```

#### CLI — Executar queries via AWS CLI

```bash
# 1. Iniciar query assíncrona
QUERY_ID=$(aws logs start-query \
  --log-group-names "/aws/lambda/payment-processor" \
  --start-time "$(date -u -d '1 hour ago' +%s 2>/dev/null || date -u -v-1H +%s)" \
  --end-time "$(date -u +%s)" \
  --query-string '
    filter level = "ERROR" or statusCode >= 400
    | stats count(*) as errorCount by endpoint, statusCode
    | sort errorCount desc
    | limit 10
  ' \
  --query 'queryId' \
  --output text)

echo "Query ID: $QUERY_ID"

# 2. Aguardar conclusão e obter resultados
sleep 5
aws logs get-query-results \
  --query-id "$QUERY_ID" \
  --query '{Status: status, Results: results}'

# 3. Query em múltiplos log groups simultaneamente
QUERY_ID2=$(aws logs start-query \
  --log-group-names \
      "/aws/lambda/payment-processor" \
      "/aws/lambda/order-service" \
      "/aws/lambda/notification-service" \
  --start-time "$(date -u -d '1 hour ago' +%s 2>/dev/null || date -u -v-1H +%s)" \
  --end-time "$(date -u +%s)" \
  --query-string '
    filter level = "ERROR"
    | stats count(*) as errors by @log, bin(5m)
    | sort bin desc
  ' \
  --query 'queryId' \
  --output text)

# 4. Listar queries salvas no console
aws logs describe-query-definitions \
  --query 'queryDefinitions[*].{Name:name,QueryString:queryString}'

# 5. Salvar uma query reutilizável
aws logs put-query-definition \
  --name "payment-error-rate-by-endpoint" \
  --log-group-names "/aws/lambda/payment-processor" \
  --query-string '
    filter level = "ERROR" or statusCode >= 400
    | stats count(*) as errors, count(*) / 1 as errorRate by endpoint, bin(5m)
    | sort bin desc
  '

# 6. Verificar custo aproximado de scaneamento
# CloudWatch Logs → Insights → Histórico de queries no console
# Ou via CloudWatch Metrics:
aws cloudwatch get-metric-statistics \
  --namespace AWS/Logs \
  --metric-name DataScannedInBytes \
  --start-time "$(date -u -d '1 day ago' '+%Y-%m-%dT%H:%M:%SZ' 2>/dev/null || date -u -v-1d '+%Y-%m-%dT%H:%M:%SZ')" \
  --end-time "$(date -u '+%Y-%m-%dT%H:%M:%SZ')" \
  --period 86400 \
  --statistics Sum \
  --query 'Datapoints[0].Sum'
```

#### Dashboard CDK — Widget com Logs Insights query

```python
from aws_cdk import (
    Stack, Duration,
    aws_cloudwatch as cw,
)
from constructs import Construct


class LogsInsightsDashboardStack(Stack):
    def __init__(self, scope: Construct, construct_id: str, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        dashboard = cw.Dashboard(
            self, "ApiDashboard",
            dashboard_name="api-insights",
        )

        # Widget de Logs Insights — timeseries de erros por endpoint
        dashboard.add_widgets(
            cw.LogQueryWidget(
                title="Error Rate by Endpoint (5m)",
                log_group_names=["/aws/lambda/payment-processor"],
                view=cw.LogQueryVisualizationType.LINE,
                width=24,
                height=6,
                query_lines=[
                    "filter level = 'ERROR' or statusCode >= 400",
                    "| stats count(*) as errors by endpoint, bin(5m)",
                    "| sort bin desc",
                ],
            ),
        )

        dashboard.add_widgets(
            # Widget de tabela — top erros
            cw.LogQueryWidget(
                title="Top Error Messages (last 1h)",
                log_group_names=["/aws/lambda/payment-processor"],
                view=cw.LogQueryVisualizationType.TABLE,
                width=12,
                height=6,
                query_lines=[
                    "filter level = 'ERROR'",
                    "| stats count(*) as occurrences by message",
                    "| sort occurrences desc",
                    "| limit 10",
                ],
            ),
            # Widget de barras — latência p99 por endpoint
            cw.LogQueryWidget(
                title="p99 Latency by Endpoint",
                log_group_names=["/aws/lambda/payment-processor"],
                view=cw.LogQueryVisualizationType.BAR,
                width=12,
                height=6,
                query_lines=[
                    "filter ispresent(latencyMs)",
                    "| stats pct(latencyMs, 99) as p99 by endpoint",
                    "| sort p99 desc",
                    "| limit 10",
                ],
            ),
        )
```

---

## Armadilhas comuns

**1. Custo por GB escaneado — queries sobre log groups grandes**  
[FATO] Logs Insights cobra $0.005/GB escaneado (us-east-1). Um log group de 100 GB com retention de 30 dias pode custar $0.50 por query se o intervalo temporal não for restrito. Sempre use o menor intervalo temporal que responda à sua pergunta. Para análises históricas frequentes, considere exportar logs para S3 e usar Athena (mais barato para grandes volumes).

**2. `parse` em logs JSON é desnecessário e lento**  
[FATO] Logs Insights já indexa automaticamente campos de primeiro nível de logs JSON. Usar `parse @message "\"latency\": *" as latency` em um log JSON é redundante e usa mais CPU. Para logs JSON, use os campos diretamente.

**3. `sort` e `limit` devem vir após o último `stats`**  
[FATO] A linguagem exige que `sort` e `limit` apareçam após o último comando `stats`. Colocar `sort` antes de `stats` gera erro de sintaxe ou resultados incorretos.

**4. Campos com caracteres especiais requerem backticks**  
[FATO] Campos que contêm hífens, pontos ou começam com número precisam de backticks. Exemplo: um log com `"detail-type"` deve ser referenciado como `` `detail-type` `` na query. Campos como `requestParameters.userName` (nested) são acessados com dot-notation diretamente.

**5. Queries em múltiplos log groups retornam campo `@log`**  
[FATO] Ao consultar múltiplos log groups, o campo `@log` identifica de qual log group veio cada evento — `account-id:log-group-name`. Sem filtrar por `@log`, você pode misturar inadvertidamente eventos de diferentes serviços em uma mesma agregação.

**6. `count_distinct` é aproximado para alta cardinalidade**  
[FATO] Para campos com muitos valores únicos (ex: `userId`), `count_distinct` usa HyperLogLog internamente e pode ter erro de estimativa de ~2-3%. Para contagens exatas de alta cardinalidade, exporte os dados para Athena.

---

## Exercício de reflexão

Você tem um log group `/aws/lambda/checkout-service` com logs JSON estruturados seguindo o padrão do Powertools Logger. Os campos relevantes são: `level`, `endpoint`, `statusCode`, `latencyMs`, `userId`, `cartId`, `errorCode`.

Escreva as queries para:

1. **SLA de disponibilidade por endpoint:** Retornar, para cada endpoint, o total de requests, o total de erros (statusCode >= 400 ou level = ERROR), e o percentual de sucesso — agrupado por janela de 1 hora nas últimas 24 horas. Ordene por taxa de erro descendente.

2. **Diagnóstico de usuário afetado:** Dado um `userId` específico (`u-suspicious-123`), liste em ordem cronológica todos os eventos de erro das últimas 6 horas, mostrando `@timestamp`, `endpoint`, `statusCode`, `errorCode`, e `cartId`.

3. **Parse em log legado:** O serviço antigo emite logs no formato texto: `"[2024-01-15 10:23:45] WARN /checkout/confirm - user=u-abc123 cart=cart-456 error=STOCK_DEPLETED"`. Escreva uma query com `parse` (glob ou regex) que extrai `warnUser`, `warnCart` e `warnError`, e agrega por `warnError`.

---

## Recursos para aprofundar

- [FATO] CloudWatch Logs Insights query syntax: https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax.html
- [FATO] Sample queries (Lambda, VPC Flow, CloudTrail, API Gateway): https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax-examples.html
- [FATO] Supported logs and discovered fields: https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_AnalyzeLogData-discoverable-fields.html
- [FATO] Functions reference (boolean, datetime, numeric): https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax-operations-functions.html

---

