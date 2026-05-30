# Sessão 30 — DynamoDB: single-table design, adjacency list e overloaded indexes

**Duração estimada:** 60 minutos
**Pré-requisitos:** session-029-dynamodb-access-patterns-pk-sk

---

## Objetivo

Ao final, você conseguirá implementar o padrão adjacency list para representar relacionamentos
muitos-para-muitos em uma única tabela (ex: usuários e grupos), usar composite sort keys para
queries hierárquicas (`ORDER#123#ITEM#456`), e identificar explicitamente quando single-table
design cria mais problema do que resolve (ex: times com tooling fraco, acessos altamente
imprevisíveis).

---

## Contexto

[FATO] A sessão anterior cobriu o fundamento: access patterns first, PK/SK genéricos com prefixos
de entidade, item collections. Esta sessão sobe um nível: como modelar relacionamentos
**muitos-para-muitos** (o problema mais difícil em bancos de documentos), como usar o sort key
para criar hierarquias de dados consultáveis em qualquer nível, e como um único GSI pode indexar
múltiplos tipos de entidade via overloading.

[OPINIÃO] O debate single-table vs multi-table é genuinamente aberto na comunidade DynamoDB.
Rick Houlihan (ex-AWS) popularizou o single-table como dogma por volta de 2018–2020. A própria
AWS, a partir de 2023–2024, passou a apresentar nos docs oficiais um exemplo canônico de 15
access patterns usando **multi-table design com GSIs estratégicos** — um recuo notável da posição
anterior. Hoje, o consenso prático é: single-table é ótimo quando as entidades têm alta correlação
de acesso; multi-table é mais simples quando as entidades têm ciclos de vida independentes. Não
há resposta universal.

---

## Conceitos principais

### 1. Adjacency list — modelando muitos-para-muitos sem joins

Em SQL, um relacionamento muitos-para-muitos entre `User` e `Group` usa uma tabela de junção:
`user_group(user_id, group_id)`. O DynamoDB não tem joins. A solução é o **adjacency list**: cada
nó (entidade) e cada aresta (relacionamento) vira um item na mesma tabela, usando PK para
representar o nó de origem e SK para representar o nó de destino.

```
Relacionamento muitos-para-muitos: Users ↔ Groups
  - Um usuário pertence a muitos grupos
  - Um grupo tem muitos usuários

Tradução para adjacency list:

PK              SK              Tipo de item
──────────────────────────────────────────────────────────────
USER#u1         USER#u1         item-raiz do usuário u1
USER#u1         GROUP#g10       aresta: u1 pertence ao grupo g10
USER#u1         GROUP#g20       aresta: u1 pertence ao grupo g20
USER#u2         USER#u2         item-raiz do usuário u2
USER#u2         GROUP#g10       aresta: u2 pertence ao grupo g10
GROUP#g10       GROUP#g10       item-raiz do grupo g10
GROUP#g10       USER#u1         aresta: g10 tem o membro u1
GROUP#g10       USER#u2         aresta: g10 tem o membro u2
GROUP#g20       GROUP#g20       item-raiz do grupo g20
GROUP#g20       USER#u1         aresta: g20 tem o membro u1
```

O diagrama como grafo:

```
u1 ──── g10 ──── u2
 └───── g20
```

Cada aresta é **representada duas vezes** — uma na partição do nó de origem, uma na partição
do nó de destino. Isso é **duplicação intencional** para suportar ambas as direções de query com
uma única operação:

```
"Quais grupos o usuário u1 pertence?"
  Query(PK="USER#u1", begins_with(SK, "GROUP#"))
  → retorna GROUP#g10 e GROUP#g20

"Quais usuários pertencem ao grupo g10?"
  Query(PK="GROUP#g10", begins_with(SK, "USER#"))
  → retorna USER#u1 e USER#u2
```

[FATO] A vantagem do adjacency list é que ambas as direções de traversal são O(1) operações de
Query — sem Scan, sem joins no aplicativo. A desvantagem é que **manter consistência é
responsabilidade do código**: se você adiciona a aresta `USER#u1 → GROUP#g10`, deve também
adicionar `GROUP#g10 → USER#u1`. Isso é normalmente feito em uma transação DynamoDB
(`transact_write_items`) para garantir atomicidade.

[FATO] Para relacionamentos muitos-para-muitos complexos (multi-hop: "amigos de amigos", grafos
de dependências, rankings de vizinhos), o adjacency list no DynamoDB começa a ser limitado —
cada "salto" exige uma nova Query. Para grafos profundos com consultas de múltiplos níveis em
tempo real, a documentação oficial da AWS recomenda usar **Amazon Neptune** (banco de dados de
grafos dedicado) em vez de tentar simular grafos profundos no DynamoDB.

### 2. Composite sort keys — hierarquias consultáveis em múltiplos níveis

Uma das propriedades mais poderosas do sort key é que, por ser uma string com ordenação
lexicográfica, você pode **embutir hierarquias** nele usando separadores. Isso permite fazer
queries que param em qualquer nível da hierarquia usando `begins_with`.

**Exemplo: sistema de pedidos com itens e sub-itens**

```
PK              SK                          Representa
─────────────────────────────────────────────────────────────────────
ORDER#o1        ORDER#o1                    cabeçalho do pedido o1
ORDER#o1        ITEM#i100                   item i100 do pedido o1
ORDER#o1        ITEM#i100#RETURN#r1         devolução r1 do item i100
ORDER#o1        ITEM#i100#RETURN#r2         devolução r2 do item i100
ORDER#o1        ITEM#i101                   item i101 do pedido o1
ORDER#o1        ITEM#i101#RETURN#r3         devolução r3 do item i101
ORDER#o1        SHIPMENT#s1                 envio s1 do pedido o1
ORDER#o1        SHIPMENT#s1#EVENT#e1        evento do envio (ex: saiu do CD)
ORDER#o1        SHIPMENT#s1#EVENT#e2        evento do envio (ex: em trânsito)
```

Com esse schema, **uma operação por nível de hierarquia**:

```
"Todos os itens do pedido o1":
  Query(PK="ORDER#o1", begins_with(SK, "ITEM#"))
  → retorna ITEM#i100, ITEM#i100#RETURN#r1, ITEM#i100#RETURN#r2, ITEM#i101, ...

"Só os itens (sem devoluções) do pedido o1":
  Não é possível com begins_with(SK, "ITEM#") sem FilterExpression — returns todos
  Solução: usar SK = "ITEM#i100" (nível exato) para GetItem, ou redesenhar o SK

"Só as devoluções do item i100":
  Query(PK="ORDER#o1", begins_with(SK, "ITEM#i100#RETURN#"))
  → retorna ITEM#i100#RETURN#r1, ITEM#i100#RETURN#r2

"Todos os eventos do envio s1":
  Query(PK="ORDER#o1", begins_with(SK, "SHIPMENT#s1#EVENT#"))
  → retorna EVENT#e1, EVENT#e2
```

[FATO] A limitação importante: **você não pode pular níveis** da hierarquia com `begins_with`.
Para buscar "todas as devoluções de todos os itens do pedido o1", você precisaria de uma Query
por `begins_with(SK, "ITEM#")` e depois filtrar no cliente os itens que contêm `#RETURN#` — ou
redesenhar o schema. O `begins_with` sempre começa do início da string; não é um "contains".

**Exemplo geográfico (da documentação AWS):**
```
SK hierárquico para localização:
  "BR#SP#SaoPaulo#Pinheiros#AvenidaFaria Lima"

Queries possíveis:
  begins_with(SK, "BR#")              → todos os endereços no Brasil
  begins_with(SK, "BR#SP#")           → todos no estado de São Paulo
  begins_with(SK, "BR#SP#SaoPaulo#")  → todos na cidade de São Paulo
  begins_with(SK, "BR#SP#SaoPaulo#Pinheiros#") → só o bairro Pinheiros
```

### 3. Overloaded GSI — um índice que serve múltiplos access patterns

[FATO] O DynamoDB permite até **20 GSIs por tabela** (limite padrão, aumentável via quota request).
Em single-table design com muitos tipos de entidade, 20 GSIs podem ser insuficientes — ou simplesmente
caros (cada GSI replica atributos e cobra por storage e writes). O **GSI overloading** usa a
flexibilidade do schema do DynamoDB (cada item pode ter atributos diferentes) para fazer um único
GSI servir múltiplos access patterns.

A ideia central: se diferentes tipos de entidade precisam ser consultados por atributos diferentes,
você cria atributos genéricos (`GSI1PK`, `GSI1SK`) cujos valores são preenchidos com conteúdo
semântico específico por tipo de entidade:

```
Tabela: plataforma-educacional (single-table)

Item type   PK            SK              GSI1PK          GSI1SK
──────────────────────────────────────────────────────────────────────────────
Student     STUDENT#s1    PROFILE         STUDENT         2024-03-01  ← query: todos alunos
Student     STUDENT#s1    COURSE#c10      COURSE#c10      STUDENT#s1  ← query: alunos de um curso
Course      COURSE#c10    METADATA        INSTRUCTOR#i5   COURSE#c10  ← query: cursos de um instrutor
Enrollment  COURSE#c10    STUDENT#s1      COURSE#c10      STUDENT#s1  ← (duplicata da aresta)
```

O mesmo GSI1 (com GSI1PK como PK e GSI1SK como SK) serve três patterns diferentes:

```
"Todos os alunos cadastrados (para admin)":
  Query GSI1(GSI1PK="STUDENT", GSI1SK >= "2024-01-01")

"Alunos matriculados no curso c10":
  Query GSI1(GSI1PK="COURSE#c10", begins_with(GSI1SK, "STUDENT#"))

"Cursos do instrutor i5":
  Query GSI1(GSI1PK="INSTRUCTOR#i5", begins_with(GSI1SK, "COURSE#"))
```

[FATO] O atributo `GSI1SK` com valor `"STUDENT"` e data para o primeiro pattern é um exemplo do
padrão **sparse index + chave estática como PK de GSI**. A chave estática (`"STUDENT"`) cria uma
"hot partition" potencial no GSI — se houver milhares de alunos sendo inseridos por segundo, essa
partição do GSI pode ser gargalo. Para workloads de alto throughput, adiciona-se um **shard** ao
valor: `"STUDENT#0"` a `"STUDENT#9"` aleatoriamente, e a query paralela nos 10 shards é feita
no cliente.

### 4. Quando single-table design cria mais problema do que resolve

[OPINIÃO — posição de Alex DeBrie, The DynamoDB Book] Single-table não é para todos, e aplicá-lo
dogmaticamente é um erro frequente. Os contextos onde **multi-table é a escolha mais segura**:

```
1. TIME COM TOOLING FRACO OU SDK DE ALTO NÍVEL
   ORM-style SDKs (Java DynamoDBMapper, Java Enhanced Client, boto3 high-level Resource)
   mapeiam classes para tabelas. Misturar tipos em uma tabela exige processar resultados
   heterogêneos — o código fica complexo e propenso a bugs de deserialização.
   Sintoma: "meu código de Query retorna um mix de User e Order e eu não sei qual é qual"

2. ENTIDADES COM CICLOS DE VIDA E OPERACIONAIS INDEPENDENTES
   Se você precisa fazer backup apenas de Orders (não de Users), ou aplicar TTL só em Sessions,
   ou habilitar InfrequentAccess storage class apenas em dados históricos — single-table força
   a mesma política para todos os tipos. Multi-table dá controle granular.

3. STREAMS COM PROCESSAMENTO POR TIPO
   DynamoDB Streams emite um evento para cada item modificado. Em single-table, uma Lambda que
   processa o stream recebe todos os tipos misturados e precisa filtrar por `entity_type`. Com
   Lambda event filters por atributo isso tem custo zero de invocação para os filtrados, mas
   aumenta a complexidade do código de processamento.

4. ACCESS PATTERNS ALTAMENTE IMPREVISÍVEIS OU EM CONSTANTE MUDANÇA
   Single-table exige que os access patterns sejam conhecidos e estáveis antes do design. Se o
   negócio muda frequentemente e novos patterns surgem, cada novo pattern pode exigir um novo
   GSI — e você fica limitado pelos 20 GSIs. Multi-table permite criar índices em tabelas
   específicas sem afetar o schema das outras.

5. TIMES QUE NÃO DOMINAM O MODELO MENTAL DO DYNAMODB
   Single-table com adjacency lists, overloaded GSIs e composite sort keys exige que TODOS os
   desenvolvedores do time entendam o design profundamente. Um desenvolvedor que não entende e
   adiciona um atributo no lugar errado pode quebrar queries silenciosamente.
   Multi-table tem a "estranheza" da falta de joins, mas o design por tabela é mais intuitivo.
```

[FATO] A documentação oficial da AWS (página `bp-modeling-nosql-B`, atualizada em 2024) passou
a usar um exemplo de 15 access patterns implementado com **multi-table design com GSIs
estratégicos** como abordagem canônica — representando uma mudança notável em relação ao
evangelismo de single-table dos anos anteriores. A própria AWS documenta as trade-offs de cada
abordagem em `data-modeling-foundations.html` sem declarar um vencedor universal.

[CONSENSO] O critério prático mais útil: se você frequentemente precisa buscar **dados de múltiplas
entidades em uma única chamada** (ex: perfil do usuário + seus últimos pedidos + suas sessões
ativas), single-table com item collections é claramente superior. Se as entidades raramente
são acessadas juntas, multi-table é mais simples de manter sem penalidade de performance.

---

## Exemplo prático

**Cenário:** plataforma de cursos online com relacionamento muitos-para-muitos entre Students e
Courses, e hierarquia de Course → Module → Lesson.

### Schema completo

```
Tabela: edtech-platform
PK (String) + SK (String) + GSI1PK (String) + GSI1SK (String)

PK              SK                      GSI1PK          GSI1SK          Dados extras
─────────────────────────────────────────────────────────────────────────────────────────────
STUDENT#s1      PROFILE                 STUDENT         2024-09-01      name, email
STUDENT#s1      ENROLLMENT#c10          COURSE#c10      STUDENT#s1      enrolled_at, progress
STUDENT#s1      ENROLLMENT#c20          COURSE#c20      STUDENT#s1      enrolled_at, progress
COURSE#c10      METADATA                INSTRUCTOR#i5   COURSE#c10      title, description
COURSE#c10      MODULE#m1               —               —               title, order_num
COURSE#c10      MODULE#m1#LESSON#l1     —               —               title, duration_min
COURSE#c10      MODULE#m1#LESSON#l2     —               —               title, duration_min
COURSE#c10      MODULE#m2               —               —               title, order_num
COURSE#c10      MODULE#m2#LESSON#l3     —               —               title, duration_min
COURSE#c10      ENROLLMENT#STUDENT#s1  COURSE#c10      STUDENT#s1      progress, last_seen
COURSE#c10      ENROLLMENT#STUDENT#s2  COURSE#c10      STUDENT#s2      progress, last_seen
```

### Access patterns atendidos

```
AP1: Perfil do aluno s1
  GetItem(PK="STUDENT#s1", SK="PROFILE")

AP2: Todos os cursos em que s1 está matriculado
  Query(PK="STUDENT#s1", begins_with(SK, "ENROLLMENT#"))

AP3: Todos os alunos matriculados no curso c10
  Query GSI1(GSI1PK="COURSE#c10", begins_with(GSI1SK, "STUDENT#"))

AP4: Todos os cursos do instrutor i5
  Query GSI1(GSI1PK="INSTRUCTOR#i5", begins_with(GSI1SK, "COURSE#"))

AP5: Estrutura completa do curso c10 (módulos + lições)
  Query(PK="COURSE#c10", begins_with(SK, "MODULE#"))

AP6: Só os módulos do curso c10 (sem as lições)
  Query(PK="COURSE#c10", begins_with(SK, "MODULE#"))
  + FilterExpression: "attribute_not_exists(lesson_id)"  ← custo: lê tudo, filtra cliente
  — ou redesenhar SK para módulos como "MODULE#m1#" e lições como "MODULE#m1#L#l1"

AP7: Só as lições do módulo m1 do curso c10
  Query(PK="COURSE#c10", begins_with(SK, "MODULE#m1#LESSON#"))
```

### Código Python — adjacency list com transação

```python
import boto3
from boto3.dynamodb.conditions import Key, Attr
from botocore.exceptions import ClientError
from datetime import datetime, timezone

dynamodb = boto3.resource("dynamodb", region_name="us-east-1")
table = dynamodb.Table("edtech-platform")


# Matricular aluno em curso — operação bidirecional em transação
def enroll_student(student_id: str, course_id: str) -> None:
    now = datetime.now(timezone.utc).isoformat()

    try:
        dynamodb.meta.client.transact_write_items(
            TransactItems=[
                # Aresta: Student → Course (na partição do aluno)
                {
                    "Put": {
                        "TableName": "edtech-platform",
                        "Item": {
                            "PK": {"S": f"STUDENT#{student_id}"},
                            "SK": {"S": f"ENROLLMENT#{course_id}"},
                            "GSI1PK": {"S": f"COURSE#{course_id}"},
                            "GSI1SK": {"S": f"STUDENT#{student_id}"},
                            "entity_type": {"S": "ENROLLMENT"},
                            "enrolled_at": {"S": now},
                            "progress": {"N": "0"},
                        },
                        "ConditionExpression": "attribute_not_exists(PK)",  # idempotente
                    }
                },
                # Aresta inversa: Course → Student (na partição do curso)
                {
                    "Put": {
                        "TableName": "edtech-platform",
                        "Item": {
                            "PK": {"S": f"COURSE#{course_id}"},
                            "SK": {"S": f"ENROLLMENT#STUDENT#{student_id}"},
                            "GSI1PK": {"S": f"COURSE#{course_id}"},
                            "GSI1SK": {"S": f"STUDENT#{student_id}"},
                            "entity_type": {"S": "ENROLLMENT"},
                            "enrolled_at": {"S": now},
                            "progress": {"N": "0"},
                            "last_seen": {"S": now},
                        },
                        "ConditionExpression": "attribute_not_exists(PK)",
                    }
                },
            ]
        )
    except ClientError as e:
        if e.response["Error"]["Code"] == "TransactionCanceledException":
            print("Aluno já matriculado (condition check falhou)")
        else:
            raise


# AP2: Quais cursos o aluno está matriculado?
def get_student_courses(student_id: str) -> list[dict]:
    response = table.query(
        KeyConditionExpression=(
            Key("PK").eq(f"STUDENT#{student_id}") &
            Key("SK").begins_with("ENROLLMENT#")
        )
    )
    return response.get("Items", [])


# AP3: Quais alunos estão no curso? (via GSI1)
def get_course_students(course_id: str) -> list[dict]:
    response = table.query(
        IndexName="GSI1",
        KeyConditionExpression=(
            Key("GSI1PK").eq(f"COURSE#{course_id}") &
            Key("GSI1SK").begins_with("STUDENT#")
        )
    )
    return response.get("Items", [])


# AP5: Estrutura do curso (módulos + lições em uma Query)
def get_course_structure(course_id: str) -> dict:
    response = table.query(
        KeyConditionExpression=(
            Key("PK").eq(f"COURSE#{course_id}") &
            Key("SK").begins_with("MODULE#")
        ),
        ScanIndexForward=True,  # ordem crescente → módulos e lições em ordem
    )
    items = response.get("Items", [])

    # Organizar hierarquia no cliente
    modules = {}
    for item in items:
        sk = item["SK"]
        parts = sk.split("#")

        if len(parts) == 2:
            # É um módulo: MODULE#m1
            mod_id = parts[1]
            modules[mod_id] = {**item, "lessons": []}
        elif len(parts) == 4 and parts[2] == "LESSON":
            # É uma lição: MODULE#m1#LESSON#l1
            mod_id = parts[1]
            lesson_id = parts[3]
            if mod_id in modules:
                modules[mod_id]["lessons"].append(item)

    return {"course_id": course_id, "modules": list(modules.values())}


# AP7: Lições de um módulo específico
def get_module_lessons(course_id: str, module_id: str) -> list[dict]:
    response = table.query(
        KeyConditionExpression=(
            Key("PK").eq(f"COURSE#{course_id}") &
            Key("SK").begins_with(f"MODULE#{module_id}#LESSON#")
        ),
        ScanIndexForward=True,
    )
    return response.get("Items", [])
```

### CLI — queries com GSI overloaded

```bash
# AP3: alunos matriculados no curso c10 (via GSI1)
aws dynamodb query \
  --table-name edtech-platform \
  --index-name GSI1 \
  --key-condition-expression "GSI1PK = :pk AND begins_with(GSI1SK, :prefix)" \
  --expression-attribute-values '{":pk":{"S":"COURSE#c10"}, ":prefix":{"S":"STUDENT#"}}'

# AP4: cursos do instrutor i5 (via GSI1)
aws dynamodb query \
  --table-name edtech-platform \
  --index-name GSI1 \
  --key-condition-expression "GSI1PK = :pk AND begins_with(GSI1SK, :prefix)" \
  --expression-attribute-values '{":pk":{"S":"INSTRUCTOR#i5"}, ":prefix":{"S":"COURSE#"}}'

# AP7: lições do módulo m1 do curso c10
aws dynamodb query \
  --table-name edtech-platform \
  --key-condition-expression "PK = :pk AND begins_with(SK, :prefix)" \
  --expression-attribute-values '{
    ":pk":{"S":"COURSE#c10"},
    ":prefix":{"S":"MODULE#m1#LESSON#"}
  }'
```

---

## Armadilhas comuns

### Armadilha 1: esquecer a aresta inversa no adjacency list

O erro mais comum ao implementar adjacency list é escrever apenas a aresta na direção da operação
atual. Por exemplo: ao matricular um aluno, o código cria `STUDENT#s1 → ENROLLMENT#c10`, mas
esquece de criar `COURSE#c10 → ENROLLMENT#STUDENT#s1`. Tudo funciona até o dia que alguém precisa
do AP3 ("quais alunos estão no curso?") e descobre que a partição do curso não tem os dados.
O problema é silencioso — sem erro, sem exceção, só resultado vazio. A prevenção é sempre
implementar operações bidirecionais em `transact_write_items` e testar ambas as direções de
query desde o início do desenvolvimento.

### Armadilha 2: composite sort key com begins_with não distingue níveis intermediários

Se o SK hierárquico é `MODULE#m1#LESSON#l1`, a query `begins_with(SK, "MODULE#m1#")` retorna
tanto o item do módulo (`MODULE#m1`) quanto todas as suas lições (`MODULE#m1#LESSON#l1`,
`MODULE#m1#LESSON#l2`). Para buscar "só o módulo, sem as lições", não há como expressar isso
apenas com KeyConditionExpression — você precisaria de uma query com `SK = "MODULE#m1"` (GetItem
ou Query com `=`), ou re-projetar o SK para que módulos e lições tenham prefixos distintos que
não sejam prefixo um do outro: ex, módulos com `MOD#m1` e lições com `LES#m1#l1`. Planejar
os prefixos de forma que não haja ambiguidade entre níveis da hierarquia é parte do design.

### Armadilha 3: atributo GSI_PK com valor estático em alta cardinalidade

No overloaded GSI, se você usa `GSI1PK = "STUDENT"` para indexar todos os alunos e a tabela
tem 10 milhões de alunos, essa partição do GSI atinge a capacidade de uma única partição (~10 GB,
~3.000 RCU/WCU). Writes de novos alunos ficarão limitados a ~1.000 WCU/s nessa partição — uma
hot partition clássica. O sintoma é `ProvisionedThroughputExceededException` no GSI mesmo com
a tabela principal operando normalmente. A solução é adicionar um sufixo de shard aleatório:
`GSI1PK = "STUDENT#" + str(random.randint(0, 9))`, e ao buscar todos os alunos fazer 10 queries
paralelas (shards 0–9) e consolidar no cliente.

---

## Exercício de reflexão

Você está modelando uma rede social profissional simplificada. As entidades são: User, Company,
e a relação "trabalha em" (Employment) entre User e Company. Um usuário pode ter trabalhado em
múltiplas empresas (histórico de empregos), e uma empresa tem muitos funcionários atuais e
ex-funcionários.

Os access patterns são:
- Buscar perfil de um usuário
- Listar todo o histórico de empregos de um usuário (cronológico, mais recente primeiro)
- Listar todos os funcionários atuais de uma empresa
- Listar ex-funcionários de uma empresa (status = "former")
- Buscar o emprego atual de um usuário (só o mais recente com status = "current")

Desenhe o schema: quais são os valores de PK, SK e GSI1PK/GSI1SK para cada tipo de item
(User, Company, Employment bidirecional). Explique como o composite sort key no item de
Employment pode incluir o timestamp de início do emprego para resolver o requisito "mais recente
primeiro". Identifique qual access pattern **não é possível** atender apenas com o SK e needs
um GSI — e explique por quê um FilterExpression seria inadequado para esse caso.

---

## Recursos para aprofundar

**1. Best practices for managing many-to-many relationships in DynamoDB tables**
URL: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-adjacency-graphs.html
O que você vai encontrar: o padrão adjacency list descrito com o exemplo canônico de
Invoices-Bills (muitos-para-muitos), schema da tabela base e do GSI de inversão, e o padrão
de grafo materializado para workflows de grafo em tempo real.
Por que é a fonte certa: documentação primária com o esquema exato e a motivação para o padrão.

**2. Overloading Global Secondary Indexes in DynamoDB**
URL: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-gsi-overloading.html
O que você vai encontrar: exemplo concreto de como um único GSI com atributo `Data` genérico
serve múltiplos tipos de query (por nome, por warehouse, por data de contratação) — o que a
AWS chama de "GSI overloading".
Por que é a fonte certa: a única página da documentação oficial que nomeia e explica o padrão
de overloading de GSI explicitamente.

**3. Example of modeling relational data in DynamoDB**
URL: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-modeling-nosql-B.html
O que você vai encontrar: o exemplo mais completo da AWS — 15 access patterns de um sistema de
pedidos, implementado com multi-table design e GSIs estratégicos. Inclui a tabela mapeando cada
access pattern para a operação DynamoDB correspondente.
Por que é a fonte certa: é o exemplo canônico atual da AWS (atualizado 2024) e demonstra a
virada em direção a multi-table como abordagem pragmática — importante para calibrar quando
usar single-table vs multi-table.

**4. Data Modeling foundations in DynamoDB**
URL: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/data-modeling-foundations.html
O que você vai encontrar: trade-offs explícitos e objetivos de single-table vs multi-table, com
lista de vantagens e desvantagens de cada abordagem (backup, encryption, streams, GraphQL, custo).
Por que é a fonte certa: é a fonte oficial mais equilibrada sobre o debate single vs multi-table,
sem dogmatismo em direção a nenhuma das abordagens.

**5. alexdebrie.com — DynamoDB one-to-many e single-table posts**
URLs: https://www.alexdebrie.com/posts/dynamodb-one-to-many/ e https://www.alexdebrie.com/posts/dynamodb-single-table/
O que você vai encontrar: os posts técnicos mais citados sobre modelagem DynamoDB fora da
documentação AWS. O primeiro cobre todos os padrões de relacionamento 1:N (embedding, item
collections, GSI). O segundo apresenta os argumentos a favor e contra single-table com mais
nuance do que qualquer re:Invent talk.
Por que é a fonte certa: [OPINIÃO] Alex DeBrie é o autor do The DynamoDB Book e provavelmente
o mais prolífico escritor técnico sobre DynamoDB fora da AWS. Seus posts são baseados em
experiência prática com workloads reais, não em evangelismo de um padrão único.

---

