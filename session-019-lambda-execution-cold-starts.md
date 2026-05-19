# Sessão session-019-lambda-execution-cold-starts — Lambda: execution model, cold starts e provisioned concurrency

**Duração estimada:** 60 minutos
**Pré-requisitos:** session-004-cdk-v2-setup-bootstrap

---

## Objetivo

Ao final, você conseguirá medir cold start de uma função com e sem provisioned concurrency, calcular o custo de provisioned concurrency vs on-demand para um perfil de tráfego específico, e identificar quais linguagens e tamanhos de package têm maior impacto em cold start.

---

## Contexto

[FATO] Lambda é um serviço de computação serverless onde você paga apenas pelo tempo que seu código executa — não por capacidade alocada em standby. O corolário dessa garantia é que quando não há execuções ativas, não há execução environments mantidos aquecidos. A próxima invocação que chegar quando nenhum environment está disponível precisa passar pela fase de inicialização antes de executar o handler. Esse custo de latência é chamado de **cold start**.

[FATO] Cold starts não são um defeito do Lambda — são uma consequência direta do modelo de cobrança. O trade-off é: você economiza pagando zero quando sua função não é invocada, mas a primeira invocação após um período de inatividade (ou qualquer invocação que exige um novo execution environment por escala horizontal) tem latência adicional. Para a maioria dos workloads assíncronos, esse trade-off é aceitável. Para APIs síncronas de baixa latência com tráfego esparso, pode ser problemático.

[CONSENSO] A importância do cold start é frequentemente exagerada em discussões de comunidade. [FATO] Segundo dados da AWS, cold starts ocorrem em menos de 1% das invocações na maioria dos workloads de produção. O problema real aparece em três cenários específicos: APIs com tráfego muito esparso (uma invocação a cada 15+ minutos), funções com inicialização pesada (Spring Boot Java, modelos ML), e aplicações interativas onde P99 de latência importa mais que média.

---

## Conceitos principais

### 1. O ciclo de vida do execution environment

[FATO] Um execution environment do Lambda passa por três fases:

```
┌─────────────────────────────────────────────────────────────────┐
│  FASE INIT (cold start — cobrada como duração)                 │
│                                                                 │
│  1. Download do código/layer (da origem: S3, ECR)              │
│     └─ Frequentemente chamado de "cold start" no senso estrito │
│                                                                 │
│  2. Inicialização do runtime (Node.js, Python, JVM, etc.)      │
│                                                                 │
│  3. Execução do código de inicialização estática               │
│     └─ Tudo FORA do handler: imports, conexões de DB,          │
│        carregamento de modelos, inicialização de SDKs          │
│                                                                 │
│  [sinaliza ready → Next API]                                   │
└─────────────────────────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────────┐
│  FASE INVOKE (execução do handler)                             │
│                                                                 │
│  handler(event, context) { ... }                               │
│                                                                 │
│  Cobrada por duração × memória                                 │
└─────────────────────────────────────────────────────────────────┘
              │
              │  função retorna, environment fica em standby
              ▼
┌─────────────────────────────────────────────────────────────────┐
│  FASE SHUTDOWN (eventual)                                      │
│                                                                 │
│  Lambda decide reciclar o environment (após inatividade)       │
│  Extensions têm até 2s para finalizar                          │
└─────────────────────────────────────────────────────────────────┘
```

[FATO] A fase INIT tem um limite de **10 segundos**. Se o código de inicialização (fora do handler) demorar mais de 10 segundos para completar, o Lambda tenta novamente na primeira invocação usando o timeout configurado da função. Funções com Spring Boot pesado ou carregamento de modelos ML de vários GB podem ultrapassar esse limite.

[FATO] O que acontece entre a fase INVOKE de uma invocação e a próxima é importante: o execution environment é **congelado** (CPU suspenso, memória mantida). Variáveis globais, conexões de banco e caches em memória **persistem entre invocações quentes**. Esse é o mecanismo que permite reutilização de conexões de banco de dados.

```python
# Python: conexão de banco criada UMA VEZ na inicialização estática
# Persiste entre invocações do mesmo execution environment
import boto3
import psycopg2

# Código FORA do handler: executado apenas no cold start
db_connection = psycopg2.connect(host=os.environ['DB_HOST'], ...)
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table(os.environ['TABLE_NAME'])

def handler(event, context):
    # Reutiliza db_connection e table (sem overhead de reconexão)
    result = table.get_item(Key={'id': event['id']})
    return result['Item']
```

---

### 2. Fatores que influenciam a duração do cold start

[FATO] Os principais fatores, em ordem de impacto:

**Runtime/linguagem:**

```
Tempo típico de cold start (inicialização do runtime, sem código de app):

  Python 3.x:      ~100-200ms
  Node.js 20+:     ~100-200ms
  Go (custom):     ~50-100ms   (binário estático, sem VM)
  .NET 8:          ~300-600ms  (com SnapStart: ~10ms)
  Java 21:         ~500-2000ms (com SnapStart: ~10ms)
  Java Spring Boot: 3000-8000ms (sem otimizações)

Nota: esses valores são aproximações baseadas em benchmarks da comunidade
e variam com memória alocada, tamanho do package, e carga dos datacenter.
```

[FATO] **Memória alocada** tem impacto indireto no cold start: mais memória = mais CPU proporcional (Lambda aloca CPU proporcional à memória). Dobrar a memória de 512MB para 1024MB pode reduzir o tempo de inicialização estática em até 50% para funções com inicialização CPU-intensiva.

[FATO] **Tamanho do deployment package** impacta o tempo de download/extração durante o cold start. Referências práticas:

```
Package pequeno (< 5 MB):   impacto mínimo (< 50ms adicional)
Package médio (5-50 MB):    50-200ms adicional
Package grande (> 50 MB):   200ms+ adicional
Container image (> 500 MB): pode adicionar 1-3s no primeiro pull

Mitigação: Lambda mantém cache do código por período de tempo não divulgado.
Pulls subsequentes ao mesmo código são muito mais rápidos.
```

[FATO] **VPC** historicamente era o maior multiplicador de cold start (adicionava 1-3 segundos para provisionar ENI). [FATO] Desde 2019, a AWS mudou a arquitetura de VPC para pre-provisionar ENIs. Cold starts em funções com VPC são agora comparáveis a funções sem VPC na maioria dos casos. O impacto residual ainda existe em contas com poucos execution environments de VPC criados recentemente.

---

### 3. Provisioned Concurrency

[FATO] Provisioned Concurrency (PC) mantém um número configurado de execution environments **pré-inicializados e aquecidos**, eliminando cold starts para esses environments. Quando uma invocação chega a um environment com PC, ela entra diretamente na fase INVOKE sem passar pelo INIT.

```
Sem PC:
  Invocação 1: [INIT 800ms] + [INVOKE 50ms] = 850ms de latência
  Invocação 2: [INVOKE 50ms] = 50ms (warm)

Com PC (2 environments provisionados):
  Invocação 1: [INVOKE 50ms] (environment já inicializado) = 50ms
  Invocação 2: [INVOKE 50ms] = 50ms
  Invocação 3: [INIT 800ms] + [INVOKE 50ms] = 850ms (PC esgotado → on-demand)
```

[FATO] PC **deve ser configurado em uma versão ou alias específico**, não em `$LATEST`:

```bash
# ❌ Errado: $LATEST não suporta PC
aws lambda put-provisioned-concurrency-config \
  --function-name my-function \
  --qualifier '$LATEST' \
  --provisioned-concurrent-executions 10

# ✅ Correto: versão numerada
aws lambda put-provisioned-concurrency-config \
  --function-name my-function \
  --qualifier 3           # versão 3
  --provisioned-concurrent-executions 10

# ✅ Correto: alias
aws lambda put-provisioned-concurrency-config \
  --function-name my-function \
  --qualifier prod        # alias 'prod'
  --provisioned-concurrent-executions 10
```

[FATO] **Custo do Provisioned Concurrency:**

```
Cobrança de PC (us-east-1, referência):
  $0.0000646234 por GB-segundo de PC alocado
  $0.0000097656 por GB-segundo de invocação em PC

Cobrança on-demand (para comparação):
  $0.0000200000 por GB-segundo de invocação

Exemplo: 10 PC × 1GB × 24h × 30 dias
  PC alocado:    10 × 1 × 86400 × 30 × $0.0000646234 = $1,679.08/mês
  Invocação em PC: depende do tráfego real
  
Total PC para 10 environments 1GB: ~$1,679/mês em alocação apenas!

→ PC tem custo fixo ALTO. Só é justificável se o custo do cold start
  (perda de SLA, usuários impactados) for maior que esse valor.
```

[FATO] Uma alternativa de custo mais controlado é usar **Application Auto Scaling** para ajustar o PC dinamicamente com base no horário:

```bash
# Escalar PC para 10 das 8h às 18h (horário comercial), 2 fora do horário
aws application-autoscaling register-scalable-target \
  --service-namespace lambda \
  --resource-id function:my-function:prod \
  --scalable-dimension lambda:function:ProvisionedConcurrency \
  --min-capacity 2 \
  --max-capacity 10
```

---

### 4. SnapStart: cold starts para Java e .NET

[FATO] SnapStart é uma feature do Lambda que captura um **snapshot** do execution environment após a fase INIT e o reutiliza para novas invocações, eliminando o tempo de inicialização do runtime e do código estático. Em vez de inicializar a JVM e carregar todas as classes a cada cold start, o Lambda restaura o estado de memória e disco do snapshot em milissegundos.

```
Sem SnapStart (Java Spring Boot):
  Deploy nova versão → [JVM init: 1s] + [Spring init: 5s] + [handler: 100ms]
  Cold start total: ~6.1 segundos

Com SnapStart:
  Deploy nova versão → Lambda faz INIT e tira snapshot
  Invocação cold → [restaura snapshot: ~10ms] + [handler: 100ms]
  Cold start total: ~110ms
```

[FATO] Limitações do SnapStart (maio 2026):
- Disponível apenas para Java (managed runtimes) e .NET 8+.
- Não suporta funções em container image (apenas deployment package).
- Não suporta funções com `ephemeralStorage > 512MB`.
- O snapshot é tirado por versão — cada `PublishVersion` gera um novo snapshot.
- Código que usa timestamps ou randoms no INIT pode ter comportamento inesperado ao ser restaurado (o tempo "congela" no snapshot).

[FATO] SnapStart tem custo zero de configuração. A cobrança é pelo tempo de invocação normal (não há custo de "alocação" como no PC).

---

## Exemplo prático

**Cenário:** Você tem uma API REST com Lambda + API Gateway. A função usa Node.js e faz conexão com RDS PostgreSQL. O tráfego é irregular: picos de 500 req/s durante horário comercial e quase zero à noite. O P99 de latência deve ser < 200ms.

### Medindo cold start no CloudWatch

```bash
# Filtrar invocações com Init Duration nos logs do Lambda
# (Init Duration aparece apenas em cold starts)
aws logs filter-log-events \
  --log-group-name /aws/lambda/my-api-function \
  --filter-pattern '"Init Duration"' \
  --start-time $(date -d '1 hour ago' +%s)000 \
  | jq '.events[].message' \
  | grep -o 'Init Duration: [0-9.]*'

# Saída típica:
# Init Duration: 823.45
# Init Duration: 791.12
# Init Duration: 1203.78
```

CloudWatch Insights para análise de cold starts em escala:

```sql
-- Query CloudWatch Insights: distribuição de cold starts por hora
filter @message like /Init Duration/
| parse @message "Init Duration: * ms" as initDuration
| stats
    count(*) as coldStarts,
    avg(initDuration) as avgInit,
    pct(initDuration, 95) as p95Init,
    max(initDuration) as maxInit
  by bin(1h)
| sort by bin(1h) desc
```

### Calculando custo: PC vs on-demand para um perfil de tráfego

```
Perfil de tráfego:
  8h-18h (10h/dia × 22 dias úteis = 220h/mês): 100 invocações/s
  Resto (530h/mês): 2 invocações/s

Função: 1GB memória, 100ms de execução média

On-demand (sem PC):
  Total invocações/mês:
    100 req/s × 220h × 3600s = 79,200,000 invocações em pico
    2 req/s × 530h × 3600s  =  3,816,000 invocações fora do pico
    Total: 83,016,000 invocações
  
  Custo GB-segundos: 83,016,000 × 0.1s × 1GB = 8,301,600 GB-s
  Custo invocações: $0.20/M × 83.016M = $16.60
  Custo GB-s: $0.0000200000 × 8,301,600 = $166.03
  Total on-demand: ~$182.63/mês
  (+ cold starts em ~1% das invocações = ~830,000 cold starts)

Com PC de 40 environments (pico de 100 req/s × 100ms = 10 concurrent + margem):
  PC allocation: 40 × 1GB × 720h = 28,800 GB-h = 103,680,000 GB-s
  Custo PC alocado: 103,680,000 × $0.0000646234 = $6,698/mês
  
  → PC é MUITO mais caro para esse perfil.
  
Com PC de 5 environments (garante resposta rápida para tráfego baixo):
  PC allocation: 5 × 1GB × 720h × 3600s/h = 12,960,000 GB-s
  Custo PC: 12,960,000 × $0.0000646234 = $837/mês
  
  → Ainda mais caro que on-demand. PC só compensa se o SLA
    exigir que TODOS os cold starts sejam eliminados, não apenas
    os de baixo tráfego.

Conclusão para esse perfil:
  Melhor estratégia: otimizar o código estático para reduzir cold start
  (conexão lazy de DB, remover dependências não usadas)
  + aceitar cold starts ocasionais em troca de custo 10x menor.
```

### CDK com Provisioned Concurrency via alias

```typescript
import { Stack, Duration } from 'aws-cdk-lib';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as appscaling from 'aws-cdk-lib/aws-applicationautoscaling';

// Função
const fn = new lambda.Function(this, 'ApiFunction', {
  runtime: lambda.Runtime.NODEJS_22_X,
  handler: 'index.handler',
  code: lambda.Code.fromAsset('lambda'),
  memorySize: 1024,
  timeout: Duration.seconds(30),
});

// Publica versão imutável (necessária para PC)
const version = fn.currentVersion;

// Alias 'prod' aponta para a versão atual
const prodAlias = new lambda.Alias(this, 'ProdAlias', {
  aliasName: 'prod',
  version,
  provisionedConcurrentExecutions: 5,  // PC configurado no alias
});

// Auto Scaling do PC baseado em agendamento
const target = prodAlias.addAutoScaling({
  minCapacity: 2,
  maxCapacity: 20,
});

// Escala para 10 às 8h, volta para 2 às 18h (UTC-3 = UTC+3 invertido)
target.scaleOnSchedule('ScaleUpMorning', {
  schedule: appscaling.Schedule.cron({ hour: '11', minute: '0' }), // 8h BRT
  minCapacity: 10,
});
target.scaleOnSchedule('ScaleDownEvening', {
  schedule: appscaling.Schedule.cron({ hour: '21', minute: '0' }), // 18h BRT
  minCapacity: 2,
});
```

---

## Armadilhas comuns

### Armadilha 1: Conexões de banco de dados criadas dentro do handler

**O erro:** O código cria uma nova conexão de banco a cada invocação:

```python
def handler(event, context):
    # ❌ Conexão criada DENTRO do handler
    conn = psycopg2.connect(host=DB_HOST, ...)
    result = conn.execute("SELECT ...")
    conn.close()
    return result
```

**Por que acontece:** O desenvolvedor não conhece o modelo de execution environment ou está preocupado com conexões "stale" em ambientes warm.

**O custo:** Uma conexão TCP + TLS para RDS leva 50-200ms. Para uma função que executa em 10ms, isso é um overhead de 500-2000%. Com 1000 invocações/minuto, isso cria e destrói 1000 conexões/minuto no banco, potencialmente esgotando o max_connections do RDS.

**Como evitar:** Crie conexões no escopo global (fora do handler). Use RDS Proxy para gerenciar o pool de conexões no lado do banco — ele agrupa as conexões de múltiplos execution environments em um pool menor para o RDS.

```python
# ✅ Conexão criada FORA do handler (inicialização estática)
conn = None

def get_connection():
    global conn
    if conn is None or conn.closed:
        conn = psycopg2.connect(host=DB_HOST, ...)
    return conn

def handler(event, context):
    c = get_connection()
    result = c.execute("SELECT ...")
    return result
```

---

### Armadilha 2: Provisioned Concurrency em `$LATEST`

**O erro:** O pipeline configura PC na versão `$LATEST` da função:

```bash
aws lambda put-provisioned-concurrency-config \
  --function-name my-function \
  --qualifier '$LATEST' \   # ← isso falha!
  --provisioned-concurrent-executions 5
```

**Por que acontece:** `$LATEST` é um alias especial que aponta para o código mais recente. É conveniente para desenvolvimento mas não suporta PC porque é mutável — o Lambda não pode manter um snapshot de algo que muda a cada deploy.

**Como reconhecer:** Erro `InvalidParameterValueException: Provisioned Concurrency Configurations are not supported on unpublished versions`. O cold start continua acontecendo mesmo depois de configurar PC.

**Como evitar:** Sempre publique uma versão (`PublishVersion`) antes de configurar PC, e aplique PC na versão numerada ou em um alias que aponte para ela. Em CDK, use `fn.currentVersion` que publica automaticamente uma versão imutável.

---

### Armadilha 3: Imports globais desnecessários aumentando cold start

**O erro:** A função importa bibliotecas completas quando só usa uma função de cada:

```python
# ❌ Importa o SDK inteiro na inicialização
import boto3
import pandas as pd
import numpy as np
from PIL import Image

def handler(event, context):
    # Só usa S3 e uma operação de JSON
    s3 = boto3.client('s3')
    # pandas, numpy, PIL nunca são usados nesta função
```

**Por que acontece:** Copiar código de outro lugar sem revisar imports, ou manter dependências de uma versão anterior da função que foi simplificada.

**O custo:** `pandas` + `numpy` juntos podem adicionar 300-500ms ao cold start apenas pelo import. Uma função que executa em 50ms pode ter cold start de 800ms por causa de imports não utilizados.

**Como evitar:** Audite os imports regularmente. Use imports lazy (dentro do handler) para bibliotecas usadas raramente. Para Node.js, use bundlers (esbuild via `NodejsFunction` do CDK) que fazem tree-shaking e eliminam código não utilizado do bundle final.

---

## Exercício de reflexão

Você tem três funções Lambda com perfis distintos:

1. **auth-validator**: Node.js, 128MB, executa em 5ms, invocada 10.000 vezes/minuto de forma constante 24h/dia. Cold start atual: 200ms.

2. **report-generator**: Java Spring Boot, 3GB, executa em 8s, invocada 2 vezes/hora, apenas durante horário comercial. Cold start atual: 6s.

3. **image-resizer**: Python, 512MB, executa em 300ms, invocada em bursts de 0 para 500 req/s em menos de 1 minuto (campanhas de marketing imprevisíveis). Cold start atual: 400ms.

Para cada função, decida: o cold start é um problema real no contexto de uso? Se sim, qual estratégia você usaria — otimização de código, Provisioned Concurrency, SnapStart, ou outra abordagem? Calcule (aproximadamente) o custo mensal de cada estratégia escolhida e compare com o custo de não fazer nada (quantos usuários afetados, qual impacto no SLA).

---

## Recursos para aprofundar

### 1. Understanding the Lambda execution environment lifecycle
**URL:** https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtime-environment.html
**O que encontrar:** Descrição detalhada das fases Init, Invoke e Shutdown, o limite de 10 segundos da fase Init, o comportamento de reuse de execution environments, e como extensões interagem com o ciclo de vida.
**Por que é a fonte certa:** É a documentação primária do modelo de execução — base para entender todos os outros conceitos desta sessão.

### 2. Configuring provisioned concurrency for a function
**URL:** https://docs.aws.amazon.com/lambda/latest/dg/provisioned-concurrency.html
**O que encontrar:** Como configurar PC em versões e aliases, a diferença de cobrança entre PC alocado e invocação em PC, como o comportamento muda quando o PC é esgotado (fallback para on-demand), e como usar Application Auto Scaling para ajustar PC dinamicamente.
**Por que é a fonte certa:** É a referência oficial do feature, com todos os detalhes de configuração e custo.

### 3. Optimizing static initialization
**URL:** https://docs.aws.amazon.com/lambda/latest/dg/static-initialization.html
**O que encontrar:** Boas práticas para reduzir o tempo de inicialização estática (fora do handler): lazy initialization, cache de objetos pesados, medição com `Init Duration` nos logs, e exemplos por linguagem.
**Por que é a fonte certa:** É o guia prático de otimização de cold start sem necessidade de PC — a primeira linha de defesa antes de considerar provisioned concurrency.

---

