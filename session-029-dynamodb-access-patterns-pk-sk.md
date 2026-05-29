# Sessão 29 — DynamoDB: access patterns first e PK/SK genéricos

**Duração estimada:** 60 minutos
**Pré-requisitos:** session-020-lambda-event-source-mappings

---

## Objetivo

Ao final, você conseguirá listar todos os access patterns de uma aplicação antes de escrever
qualquer schema, projetar PK e SK genéricos para suportar múltiplos tipos de entidade
(ex: PK = `USER#123`, SK = `PROFILE`), e implementar GetItem por PK e Query por SK prefix em
código.

---

## Contexto

[FATO] O DynamoDB é um banco de dados NoSQL chave-valor e de documentos, totalmente gerenciado
pela AWS, que provê latências de um dígito em milissegundos em qualquer escala. Ele foi
originalmente descrito no paper "Dynamo: Amazon's Highly Available Key-value Store" (2007) e se
tornou um dos serviços mais usados na AWS por aplicações que precisam de throughput previsível sem
os custos operacionais de um banco relacional.

[FATO] A diferença fundamental entre modelar para SQL e modelar para DynamoDB não é técnica — é
filosófica. Em SQL, você normaliza os dados primeiro e escreve as queries depois ("queries follow
the data"). Em DynamoDB, é o inverso: você documenta as queries (access patterns) primeiro e
projeta o schema para servir exatamente essas queries ("data follows the queries"). Começar pelo
schema no DynamoDB quase sempre resulta em designs que exigem Scans caros ou múltiplas queries
onde uma seria suficiente.

[OPINIÃO] Rick Houlihan (ex-AWS, principal evangelista de DynamoDB) articula isso como: "If you
don't know your access patterns before you design your DynamoDB table, you're going to have a bad
time." — essa posição é consenso na comunidade de DynamoDB, documentada em inúmeras re:Invent
talks e no livro The DynamoDB Book de Alex DeBrie.

---

## Conceitos principais

### 1. O paradigma "access patterns first" — por que queries antes do schema

Em um banco relacional, o processo de design é: entidades → normalização → tabelas → índices.
As queries são escritas depois, e o otimizador de query (query planner) decide como executá-las
eficientemente usando índices, joins e planos de execução dinâmicos.

O DynamoDB não tem query planner. Cada operação de leitura deve especificar exatamente como
chegar aos dados: via chave primária (GetItem), via partição com filtro no sort key (Query), ou
via varredura completa da tabela (Scan). O Scan é a única operação que "acha" dados sem saber
onde estão — e é proibitivamente cara em tabelas grandes.

[FATO] O processo correto para DynamoDB é:

```
Passo 1: Identificar entidades do domínio
         (ex: User, Order, Product, Review, Session)

Passo 2: Identificar os relacionamentos entre entidades
         (ex: User tem muitos Orders; Order tem muitos OrderItems)

Passo 3: Documentar TODOS os access patterns necessários
         (ex: "buscar usuário por ID", "listar pedidos de um usuário",
          "buscar pedidos de um usuário por data", "buscar itens de um pedido")

Passo 4: SOMENTE ENTÃO projetar PK, SK e índices secundários
         para atender cada access pattern com uma única operação
```

[FATO] Um access pattern documenta: **quem** está lendo, **o que** está lendo, e **quais
filtros/ordenações** são necessários. Uma tabela tipicamente tem 10–30 access patterns. Cada
pattern se torna uma operação DynamoDB — GetItem, Query ou (raramente) Scan.

**Exemplo de lista de access patterns para e-commerce:**

| # | Access Pattern | Operação DynamoDB |
|---|---------------|-------------------|
| AP1 | Buscar usuário por ID | GetItem |
| AP2 | Listar todos os pedidos de um usuário | Query |
| AP3 | Listar pedidos de um usuário nos últimos 30 dias | Query (BETWEEN no SK) |
| AP4 | Buscar pedido específico por ID | GetItem |
| AP5 | Listar itens de um pedido | Query (begins_with no SK) |
| AP6 | Buscar produto por ID | GetItem |
| AP7 | Listar reviews de um produto | Query |
| AP8 | Buscar review específica de um usuário em um produto | GetItem |

Essa lista deve ser **completa e estável** antes de começar o design. Adicionar um access pattern
novo depois de criar a tabela geralmente exige criar um novo índice secundário (Global Secondary
Index — GSI), o que tem custo de storage e write throughput.

### 2. Partition key e sort key — como o DynamoDB armazena e acessa dados

[FATO] O DynamoDB tem dois tipos de chave primária:

```
Chave simples (simple primary key):
  Apenas partition key (PK)
  → identifica unicamente um item
  → só suporta GetItem (busca por valor exato)

Chave composta (composite primary key):
  Partition key (PK) + Sort key (SK)
  → PK + SK juntos identificam unicamente um item
  → múltiplos itens podem ter o mesmo PK, com SKs diferentes
  → suporta GetItem (PK + SK exatos) e Query (PK exato + condição no SK)
```

[FATO] O DynamoDB usa o valor da partition key como entrada de uma função hash interna para
determinar em qual **partição física** o item será armazenado. Itens com o mesmo valor de PK são
armazenados na mesma partição, ordenados pelo valor do SK. Esse agrupamento é chamado de
**item collection**.

```
Partição física para PK = "USER#123":
  ┌──────────────┬──────────────────────────────────────────┐
  │ PK           │ SK                   │ outros atributos  │
  ├──────────────┼──────────────────────┼───────────────────┤
  │ USER#123     │ ORDER#2024-01-15#001  │ total: 150.00    │
  │ USER#123     │ ORDER#2024-01-20#002  │ total: 89.90     │
  │ USER#123     │ ORDER#2024-03-05#003  │ total: 230.00    │
  │ USER#123     │ PROFILE              │ name: João Silva  │
  │ USER#123     │ SESSION#abc123       │ expires: ...      │
  └──────────────┴──────────────────────┴───────────────────┘
  ↑ todos os itens com PK="USER#123" ficam juntos, ordenados por SK
```

[FATO] A operação **Query** permite buscar todos os itens de uma partição (PK fixo) e
opcionalmente filtrar por SK usando condições de range. Como os itens são armazenados ordenados
pelo SK, essas operações são O(log n) no número de itens da partição — eficientes e previsíveis.

[FATO] O sort key tem tipo — String, Number, ou Binary. Para Strings, a ordenação é **lexicográfica
(UTF-8)**. Isso significa que `"2024-01-15"` ordena corretamente como data se estiver em formato
ISO 8601 (ano-mês-dia), mas `"15/01/2024"` (dia/mês/ano) não ordena corretamente como data.
Usar timestamps em formato ISO 8601 ou Unix epoch no SK é uma prática canônica.

### 3. PK e SK genéricos — o padrão de nomes e prefixos de entidade

[FATO] Em designs de single-table, é prática padrão usar nomes genéricos `PK` e `SK` para os
atributos de chave primária, em vez de nomes semânticos como `userId` ou `orderId`. Isso porque
a mesma coluna `PK` armazenará valores de tipos completamente diferentes (`USER#123`,
`ORDER#456`, `PRODUCT#789`) — um nome semântico seria enganoso.

**Prefixos de entidade** servem para:

```
1. Distinguir tipos diferentes de entidade na mesma tabela
   USER#123 vs ORDER#123  →  evita colisões de ID entre tipos

2. Permitir Query por tipo usando begins_with
   SK begins_with "ORDER#"  →  retorna apenas pedidos do usuário

3. Documentar o intent do schema para outros desenvolvedores
   PK = "USER#abc123"  →  imediatamente legível o que esse item representa

4. Prevenir bugs de "tipo errado"
   Se GetItem retorna um item com PK="ORDER#456" mas você esperava USER,
   o prefixo deixa o erro óbvio
```

**Convenção de prefixo:**

```
Entidade    PK value          SK value
─────────────────────────────────────────────────────────────
User        USER#<userId>     PROFILE
Order       USER#<userId>     ORDER#<orderId>
OrderItem   ORDER#<orderId>   ITEM#<productId>
Product     PRODUCT#<pid>     METADATA
Review      PRODUCT#<pid>     REVIEW#<userId>
Session     USER#<userId>     SESSION#<sessionId>
```

[FATO] O separador `#` é convencional mas arbitrário — você pode usar qualquer caractere que não
apareça nos IDs. O `#` é amplamente usado porque é fácil de ler e não causa problemas em URLs ou
JSON.

[FATO] Quando PK e SK têm o mesmo valor (ex: `PK = "USER#123"`, `SK = "USER#123"`), o item é
chamado de **item raiz** (root item) da partição. Esse padrão é usado quando você quer garantir
que sempre existe um item "cabeçalho" para cada entidade, mesmo quando outros itens da coleção
(pedidos, sessões) ainda não foram criados.

### 4. Item collections — agrupamento e localidade de dados

[FATO] Uma **item collection** é o conjunto de todos os itens que compartilham o mesmo valor de
partition key. É a unidade fundamental de "localidade de dados" no DynamoDB — todos os itens de
uma coleção ficam na mesma partição física e podem ser recuperados com uma única operação Query.

```
Item collection para PK = "USER#123":
  → PROFILE (dados do usuário)
  → ORDER#2024-01-15#001 (pedido de janeiro)
  → ORDER#2024-01-20#002 (pedido de janeiro)
  → ORDER#2024-03-05#003 (pedido de março)
  → SESSION#abc123 (sessão ativa)

Custo de recuperar perfil + todos os pedidos:
  SQL: JOIN entre tabela users e orders → 2 queries ou 1 com JOIN
  DynamoDB: 1 Query com PK="USER#123" → 1 operação, 0.5 RCU (se < 4KB)
```

[FATO] O limite de tamanho de uma item collection é **10 GB por valor de partition key** (apenas
para tabelas com LSI — Local Secondary Index). Para tabelas sem LSI, não há limite de tamanho por
partition key.

[FATO] Um item individual no DynamoDB tem limite de **400 KB**. Atributos muito grandes (ex:
imagens, PDFs, blobs) devem ser armazenados no S3, com o DynamoDB armazenando apenas a referência
(URL ou key do S3).

### 5. Operações GetItem e Query — implementação e semântica

[FATO] **GetItem** é a operação mais eficiente do DynamoDB — O(1), sempre lê exatamente 1 item
dado PK + SK. Nunca faz Scan. Custo: 0.5 RCU por 4KB em eventual consistency ou 1 RCU em strong
consistency.

[FATO] **Query** requer PK exato e opcionalmente uma condição no SK. Suporta os seguintes
operadores no sort key:

```
=           → valor exato
<, <=       → menor (ou igual) a um valor
>, >=       → maior (ou igual) a um valor
BETWEEN x AND y → valor entre x e y (inclusivo)
begins_with(SK, "prefix") → SK começa com o prefixo
```

[FATO] O Query retorna resultados **ordenados pelo SK** em ordem crescente por padrão (lexicográfica
para String, numérica para Number). Para ordem decrescente, use `ScanIndexForward=False`.

[FATO] Uma única chamada Query retorna no máximo **1 MB de dados**. Se a coleção tem mais de 1 MB,
a resposta inclui `LastEvaluatedKey` — um token de paginação. Para buscar todos os resultados, é
necessário fazer chamadas paginadas até `LastEvaluatedKey` ser `null`.

**FilterExpression vs KeyConditionExpression:**

[FATO] Essa distinção é crítica para entender custo no DynamoDB:

```
KeyConditionExpression:
  → aplicado ANTES de ler os dados
  → usa PK e SK para identificar quais itens ler
  → você paga apenas pelos dados que correspondem à condição de chave

FilterExpression:
  → aplicado DEPOIS de ler os dados (no lado do DynamoDB, mas após consumir RCUs)
  → filtra atributos não-chave
  → você PAGA pelos dados lidos, mesmo os filtrados de volta
  → útil para refinar resultados, mas NÃO reduz custo de leitura
```

Exemplo: Query com `PK = "USER#123"` retorna 100 itens (500 KB). Se um `FilterExpression` filtra
90 deles, você pagou por 500 KB mas recebeu apenas 50 KB de resultados. Para evitar isso, o
filtro deve sempre estar no SK (via `begins_with` ou `BETWEEN`).

---

## Exemplo prático

**Cenário:** plataforma de blog com usuários, posts e comentários. Access patterns levantados:

```
AP1: Buscar usuário por userId
AP2: Buscar post por postId
AP3: Listar todos os posts de um usuário (mais recente primeiro)
AP4: Listar posts de um usuário no último mês
AP5: Listar comentários de um post
AP6: Buscar comentário específico (postId + userId)
AP7: Buscar perfil + últimos 5 posts do usuário (transação)
```

**Schema projetado:**

```
Tabela: blog-single-table
Chave primária composta: PK (String) + SK (String)

PK value             SK value                    Representa
──────────────────────────────────────────────────────────────────────
USER#u123            PROFILE                     Perfil do usuário u123
USER#u123            POST#2024-03-15T10:00:00    Post de 15/03 do u123
USER#u123            POST#2024-03-20T14:30:00    Post de 20/03 do u123
USER#u123            POST#2024-04-01T09:15:00    Post de 01/04 do u123
POST#p456            METADATA                    Metadados do post p456
POST#p456            COMMENT#u789                Comentário do user u789
POST#p456            COMMENT#u321                Comentário do user u321
```

**Como cada access pattern é atendido:**

```
AP1 (buscar usuário):
  GetItem(PK="USER#u123", SK="PROFILE")

AP2 (buscar post):
  GetItem(PK="POST#p456", SK="METADATA")

AP3 (listar posts de usuário, mais recente primeiro):
  Query(PK="USER#u123",
        KeyConditionExpression="PK = :pk AND begins_with(SK, :prefix)",
        ExpressionAttributeValues={":pk": "USER#u123", ":prefix": "POST#"},
        ScanIndexForward=False)   ← ordem decrescente (mais recente primeiro)

AP4 (posts do último mês — abril 2024):
  Query(PK="USER#u123",
        KeyConditionExpression="PK = :pk AND SK BETWEEN :start AND :end",
        ExpressionAttributeValues={
          ":pk": "USER#u123",
          ":start": "POST#2024-04-01",
          ":end": "POST#2024-04-30T23:59:59"
        })

AP5 (comentários de um post):
  Query(PK="POST#p456",
        KeyConditionExpression="PK = :pk AND begins_with(SK, :prefix)",
        ExpressionAttributeValues={":pk": "POST#p456", ":prefix": "COMMENT#"})

AP6 (comentário específico):
  GetItem(PK="POST#p456", SK="COMMENT#u789")
```

### Implementação Python com boto3

```python
import boto3
from boto3.dynamodb.conditions import Key
from datetime import datetime, timedelta, timezone

dynamodb = boto3.resource("dynamodb", region_name="us-east-1")
table = dynamodb.Table("blog-single-table")


# AP1: Buscar usuário por ID (GetItem — O(1), sem Scan)
def get_user(user_id: str) -> dict | None:
    response = table.get_item(
        Key={
            "PK": f"USER#{user_id}",
            "SK": "PROFILE",
        },
        ConsistentRead=False,  # eventual consistency (0.5 RCU) — adequado para leitura de perfil
    )
    return response.get("Item")


# AP3: Listar todos os posts de um usuário, do mais recente para o mais antigo
def list_user_posts(user_id: str, limit: int = 20) -> list[dict]:
    response = table.query(
        KeyConditionExpression=Key("PK").eq(f"USER#{user_id}") & Key("SK").begins_with("POST#"),
        ScanIndexForward=False,   # ordem decrescente → mais recente primeiro
        Limit=limit,
    )
    return response.get("Items", [])


# AP4: Posts de um usuário dentro de um range de datas
def list_user_posts_in_range(user_id: str, start: datetime, end: datetime) -> list[dict]:
    # ISO 8601 com timezone UTC para ordenação lexicográfica correta
    start_sk = "POST#" + start.strftime("%Y-%m-%dT%H:%M:%S")
    end_sk = "POST#" + end.strftime("%Y-%m-%dT%H:%M:%S")

    response = table.query(
        KeyConditionExpression=(
            Key("PK").eq(f"USER#{user_id}") &
            Key("SK").between(start_sk, end_sk)
        ),
        ScanIndexForward=False,
    )
    return response.get("Items", [])


# AP5: Comentários de um post com paginação
def list_post_comments(post_id: str) -> list[dict]:
    all_items = []
    last_key = None

    while True:
        kwargs = {
            "KeyConditionExpression": (
                Key("PK").eq(f"POST#{post_id}") &
                Key("SK").begins_with("COMMENT#")
            ),
        }
        if last_key:
            kwargs["ExclusiveStartKey"] = last_key

        response = table.query(**kwargs)
        all_items.extend(response.get("Items", []))
        last_key = response.get("LastEvaluatedKey")

        if not last_key:
            break

    return all_items


# Criar um usuário (PutItem)
def create_user(user_id: str, name: str, email: str) -> None:
    table.put_item(
        Item={
            "PK": f"USER#{user_id}",
            "SK": "PROFILE",
            "entity_type": "USER",         # atributo auxiliar para clareza
            "user_id": user_id,
            "name": name,
            "email": email,
            "created_at": datetime.now(timezone.utc).isoformat(),
        },
        ConditionExpression="attribute_not_exists(PK)",  # evita sobrescrever usuário existente
    )


# Criar um post (SK inclui timestamp para ordenação cronológica)
def create_post(user_id: str, post_id: str, title: str, body: str) -> None:
    now = datetime.now(timezone.utc)
    timestamp = now.strftime("%Y-%m-%dT%H:%M:%S")

    # Armazena na partição do usuário (para AP3/AP4)
    table.put_item(
        Item={
            "PK": f"USER#{user_id}",
            "SK": f"POST#{timestamp}",
            "entity_type": "POST",
            "post_id": post_id,
            "title": title,
            "body": body,
            "created_at": now.isoformat(),
        }
    )

    # Armazena também na partição do post (para AP2 e AP5)
    table.put_item(
        Item={
            "PK": f"POST#{post_id}",
            "SK": "METADATA",
            "entity_type": "POST",
            "post_id": post_id,
            "user_id": user_id,
            "title": title,
            "created_at": now.isoformat(),
        }
    )
```

### CLI — consultas básicas

```bash
# AP1: GetItem — buscar usuário
aws dynamodb get-item \
  --table-name blog-single-table \
  --key '{"PK": {"S": "USER#u123"}, "SK": {"S": "PROFILE"}}' \
  --consistent-read

# AP3: Query — posts do usuário, mais recente primeiro
aws dynamodb query \
  --table-name blog-single-table \
  --key-condition-expression "PK = :pk AND begins_with(SK, :prefix)" \
  --expression-attribute-values '{":pk":{"S":"USER#u123"}, ":prefix":{"S":"POST#"}}' \
  --scan-index-forward false \
  --limit 10

# AP4: Query — posts entre duas datas
aws dynamodb query \
  --table-name blog-single-table \
  --key-condition-expression "PK = :pk AND SK BETWEEN :start AND :end" \
  --expression-attribute-values '{
    ":pk":{"S":"USER#u123"},
    ":start":{"S":"POST#2024-04-01"},
    ":end":{"S":"POST#2024-04-30T23:59:59"}
  }'

# AP5: Query — comentários de um post
aws dynamodb query \
  --table-name blog-single-table \
  --key-condition-expression "PK = :pk AND begins_with(SK, :prefix)" \
  --expression-attribute-values '{":pk":{"S":"POST#p456"}, ":prefix":{"S":"COMMENT#"}}'
```

---

## Armadilhas comuns

### Armadilha 1: usar FilterExpression como substituto de KeyConditionExpression

Uma das confusões mais frequentes: desenvolvedores criam um schema sem SK discriminante e depois
usam `FilterExpression` para filtrar por tipo de entidade. Por exemplo: tabela com `PK = userId`
e um atributo `type = "ORDER"` ou `type = "PROFILE"`, depois query com
`FilterExpression="type = ORDER"`. O DynamoDB lê **todos os itens da partição**, paga por todos,
e descarta os que não passam no filtro. Em uma partição com 10.000 itens, você paga por 10.000
leituras e recebe 500. A correção é usar `begins_with(SK, "ORDER#")` na `KeyConditionExpression`
— o DynamoDB então só lê os itens com SK correto, e você paga apenas pelo que recebe.

### Armadilha 2: timestamps no SK com fuso horário inconsistente

Se parte do código armazena timestamps em UTC e outra parte armazena em horário local, a ordenação
lexicográfica quebra: `"POST#2024-01-15T10:00:00-03:00"` e `"POST#2024-01-15T13:00:00Z"` são o
mesmo instante, mas lexicograficamente diferentes, e vão aparecer separados em uma Query com
`BETWEEN`. A regra é: **sempre UTC, sempre ISO 8601, sempre com precisão consistente**. Usar
`datetime.now(timezone.utc).strftime("%Y-%m-%dT%H:%M:%S")` em Python garante uniformidade.
Alternativamente, armazene Unix timestamps como Number — `Key("SK").between(1705316400, 1705320000)`
funciona corretamente com o tipo Number.

### Armadilha 3: item muito grande por armazenar coleções embutidas no item

O limite de 400 KB por item é generoso para a maioria dos casos, mas é fácil de atingir quando
se armazena arrays crescentes como atributos de um item: comentários embutidos em um post,
histórico de estados embutido em um pedido, tags embutidas em um produto. A solução correta é
modelar cada elemento como um **item separado** na tabela (usando SK discriminante), não como
atributo de lista no item pai. Itens separados = crescimento ilimitado, leituras granulares,
atualizações sem reescrever o item inteiro. Atributos de lista = conveniente para listas pequenas
e fixas (ex: lista de 3-5 tags), problemático para listas que crescem indefinidamente.

---

## Exercício de reflexão

Você está projetando o schema DynamoDB para um sistema de suporte ao cliente. As entidades
do domínio são: Customer (cliente), Ticket (chamado de suporte), Message (mensagem dentro de um
ticket), e Agent (agente de suporte que atende tickets).

Os access patterns levantados pela equipe são:
- Buscar dados de um cliente por ID
- Listar todos os tickets abertos de um cliente
- Listar todas as mensagens de um ticket, em ordem cronológica
- Buscar ticket por ID (para qualquer cliente)
- Listar tickets atribuídos a um agente específico hoje
- Buscar o ticket mais recente de um cliente (para o painel de atendimento)

Descreva como você organizaria a tabela: quais seriam os valores de PK e SK para cada tipo de
entidade, como a escolha de timestamp no SK resolve o requisito de "listar em ordem cronológica"
e "buscar o mais recente", e qual dos access patterns acima **não pode** ser atendido apenas com
PK + SK (necessita de um índice secundário) e por quê. Explique também por que "listar tickets
atribuídos a um agente hoje" é diferente dos outros patterns em termos de modelagem.

---

## Recursos para aprofundar

**1. Data Modeling foundations in DynamoDB**
URL: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/data-modeling-foundations.html
O que você vai encontrar: comparação estruturada entre single table design e multiple table design,
com trade-offs de cada abordagem (backup, encryption, streams, GraphQL, custo) e orientações
sobre quando usar cada um.
Por que é a fonte certa: documentação primária da AWS com a posição atual (2024+) sobre single vs
multi-table, que mudou ligeiramente dos anos anteriores quando a AWS era mais dogmática sobre
single-table.

**2. First steps for modeling relational data in DynamoDB**
URL: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-modeling-nosql.html
O que você vai encontrar: o processo de identificação de access patterns com exemplo concreto de
15 patterns para um sistema de pedidos, e o framework mental para "query first, schema last".
Por que é a fonte certa: a tabela de access patterns desse documento é o exemplo canônico da AWS
para o processo de modelagem DynamoDB.

**3. Best practices for using sort keys to organize data in DynamoDB**
URL: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-sort-keys.html
O que você vai encontrar: padrão de composite sort key para dados hierárquicos
(`[país]#[região]#[estado]#[cidade]`), padrão de version control com sort key prefixes
(`v0_` para versão atual, `v1_`... para histórico).
Por que é a fonte certa: documentação primária com os dois padrões de sort key mais citados na
literatura de DynamoDB.

**4. Key condition expressions for the Query operation in DynamoDB**
URL: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Query.KeyConditionExpressions.html
O que você vai encontrar: lista completa de operadores válidos no sort key (`=`, `<`, `<=`, `>`,
`>=`, `BETWEEN`, `begins_with`), com exemplos de CLI para cada um.
Por que é a fonte certa: referência direta para implementar cada access pattern; evita a confusão
entre operadores disponíveis em KeyCondition vs FilterExpression.

**5. The DynamoDB Book — Alex DeBrie**
URL: https://www.dynamodbbook.com
O que você vai encontrar: o livro técnico mais completo sobre modelagem DynamoDB, cobrindo todos
os padrões de design (adjacency list, GSI overloading, sparse indexes, write sharding) com
exemplos completos de aplicações reais.
Por que é a fonte certa: [OPINIÃO] Alex DeBrie é considerado pela comunidade como a referência
mais acessível e prática sobre DynamoDB — mais didático que a documentação oficial, e atualizado
regularmente. Não é gratuito, mas é amplamente citado nas re:Invent talks da AWS.

---

