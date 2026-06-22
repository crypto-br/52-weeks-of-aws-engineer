# Session 30 — DynamoDB: single-table design, adjacency list and overloaded indexes

**Estimated duration:** 60 minutes
**Prerequisites:** session-029-dynamodb-access-patterns-pk-sk

---

## Objective

By the end, you will be able to implement the adjacency list pattern to represent many-to-many
relationships in a single table (e.g., users and groups), use composite sort keys for
hierarchical queries (`ORDER#123#ITEM#456`), and explicitly identify when single-table
design creates more problems than it solves (e.g., teams with weak tooling, highly
unpredictable access patterns).

---

## Context

[FACT] The previous session covered the foundation: access patterns first, generic PK/SK with entity
prefixes, item collections. This session goes up a level: how to model **many-to-many**
relationships (the hardest problem in document databases), how to use the sort key
to create queryable data hierarchies at any level, and how a single GSI can index
multiple entity types via overloading.

[OPINION] The single-table vs multi-table debate is genuinely open in the DynamoDB community.
Rick Houlihan (ex-AWS) popularized single-table as dogma around 2018–2020. AWS itself,
starting in 2023–2024, began presenting in official docs a canonical example of 15
access patterns using **multi-table design with strategic GSIs** — a notable retreat from the
previous position. Today, the practical consensus is: single-table is great when entities have high
access correlation; multi-table is simpler when entities have independent lifecycles. There
is no universal answer.

---

## Main concepts

### 1. Adjacency list — modeling many-to-many without joins

In SQL, a many-to-many relationship between `User` and `Group` uses a junction table:
`user_group(user_id, group_id)`. DynamoDB has no joins. The solution is the **adjacency list**: each
node (entity) and each edge (relationship) becomes an item in the same table, using PK to
represent the source node and SK to represent the destination node.

```
Many-to-many relationship: Users ↔ Groups
  - A user belongs to many groups
  - A group has many users

Translation to adjacency list:

PK              SK              Item type
──────────────────────────────────────────────────────────────
USER#u1         USER#u1         root item for user u1
USER#u1         GROUP#g10       edge: u1 belongs to group g10
USER#u1         GROUP#g20       edge: u1 belongs to group g20
USER#u2         USER#u2         root item for user u2
USER#u2         GROUP#g10       edge: u2 belongs to group g10
GROUP#g10       GROUP#g10       root item for group g10
GROUP#g10       USER#u1         edge: g10 has member u1
GROUP#g10       USER#u2         edge: g10 has member u2
GROUP#g20       GROUP#g20       root item for group g20
GROUP#g20       USER#u1         edge: g20 has member u1
```

The diagram as a graph:

```
u1 ──── g10 ──── u2
 └───── g20
```

Each edge is **represented twice** — once in the source node's partition, once in the
destination node's partition. This is **intentional duplication** to support both query directions with
a single operation:

```
"Which groups does user u1 belong to?"
  Query(PK="USER#u1", begins_with(SK, "GROUP#"))
  → returns GROUP#g10 and GROUP#g20

"Which users belong to group g10?"
  Query(PK="GROUP#g10", begins_with(SK, "USER#"))
  → returns USER#u1 and USER#u2
```

[FACT] The advantage of adjacency list is that both traversal directions are O(1) Query
operations — no Scan, no joins in the application. The disadvantage is that **maintaining consistency is
the code's responsibility**: if you add the edge `USER#u1 → GROUP#g10`, you must also
add `GROUP#g10 → USER#u1`. This is normally done in a DynamoDB transaction
(`transact_write_items`) to guarantee atomicity.

[FACT] For complex many-to-many relationships (multi-hop: "friends of friends", dependency graphs,
neighbor rankings), the adjacency list in DynamoDB starts to be limited —
each "hop" requires a new Query. For deep graphs with multi-level queries in
real time, the official AWS documentation recommends using **Amazon Neptune** (dedicated graph
database) instead of trying to simulate deep graphs in DynamoDB.

### 2. Composite sort keys — queryable hierarchies at multiple levels

One of the most powerful properties of the sort key is that, being a string with lexicographic
ordering, you can **embed hierarchies** in it using separators. This allows making
queries that stop at any level of the hierarchy using `begins_with`.

**Example: order system with items and sub-items**

```
PK              SK                          Represents
─────────────────────────────────────────────────────────────────────
ORDER#o1        ORDER#o1                    order o1 header
ORDER#o1        ITEM#i100                   item i100 of order o1
ORDER#o1        ITEM#i100#RETURN#r1         return r1 of item i100
ORDER#o1        ITEM#i100#RETURN#r2         return r2 of item i100
ORDER#o1        ITEM#i101                   item i101 of order o1
ORDER#o1        ITEM#i101#RETURN#r3         return r3 of item i101
ORDER#o1        SHIPMENT#s1                 shipment s1 of order o1
ORDER#o1        SHIPMENT#s1#EVENT#e1        shipment event (e.g., left warehouse)
ORDER#o1        SHIPMENT#s1#EVENT#e2        shipment event (e.g., in transit)
```

With this schema, **one operation per hierarchy level**:

```
"All items of order o1":
  Query(PK="ORDER#o1", begins_with(SK, "ITEM#"))
  → returns ITEM#i100, ITEM#i100#RETURN#r1, ITEM#i100#RETURN#r2, ITEM#i101, ...

"Only items (without returns) of order o1":
  Not possible with begins_with(SK, "ITEM#") without FilterExpression — returns all
  Solution: use SK = "ITEM#i100" (exact level) for GetItem, or redesign the SK

"Only returns of item i100":
  Query(PK="ORDER#o1", begins_with(SK, "ITEM#i100#RETURN#"))
  → returns ITEM#i100#RETURN#r1, ITEM#i100#RETURN#r2

"All events of shipment s1":
  Query(PK="ORDER#o1", begins_with(SK, "SHIPMENT#s1#EVENT#"))
  → returns EVENT#e1, EVENT#e2
```

[FACT] The important limitation: **you cannot skip levels** of the hierarchy with `begins_with`.
To search for "all returns of all items of order o1", you would need a Query
with `begins_with(SK, "ITEM#")` and then filter on the client for items containing `#RETURN#` — or
redesign the schema. `begins_with` always starts from the beginning of the string; it's not a "contains".

**Geographic example (from AWS documentation):**
```
Hierarchical SK for location:
  "BR#SP#SaoPaulo#Pinheiros#AvenidaFaria Lima"

Possible queries:
  begins_with(SK, "BR#")              → all addresses in Brazil
  begins_with(SK, "BR#SP#")           → all in the state of São Paulo
  begins_with(SK, "BR#SP#SaoPaulo#")  → all in the city of São Paulo
  begins_with(SK, "BR#SP#SaoPaulo#Pinheiros#") → only the Pinheiros neighborhood
```

### 3. Overloaded GSI — one index serving multiple access patterns

[FACT] DynamoDB allows up to **20 GSIs per table** (default limit, increasable via quota request).
In single-table design with many entity types, 20 GSIs may be insufficient — or simply
expensive (each GSI replicates attributes and charges for storage and writes). **GSI overloading** uses the
schema flexibility of DynamoDB (each item can have different attributes) to make a single
GSI serve multiple access patterns.

The central idea: if different entity types need to be queried by different attributes,
you create generic attributes (`GSI1PK`, `GSI1SK`) whose values are filled with
semantically specific content per entity type:

```
Table: educational-platform (single-table)

Item type   PK            SK              GSI1PK          GSI1SK
──────────────────────────────────────────────────────────────────────────────
Student     STUDENT#s1    PROFILE         STUDENT         2024-03-01  ← query: all students
Student     STUDENT#s1    COURSE#c10      COURSE#c10      STUDENT#s1  ← query: students in a course
Course      COURSE#c10    METADATA        INSTRUCTOR#i5   COURSE#c10  ← query: courses by instructor
Enrollment  COURSE#c10    STUDENT#s1      COURSE#c10      STUDENT#s1  ← (edge duplicate)
```

The same GSI1 (with GSI1PK as PK and GSI1SK as SK) serves three different patterns:

```
"All registered students (for admin)":
  Query GSI1(GSI1PK="STUDENT", GSI1SK >= "2024-01-01")

"Students enrolled in course c10":
  Query GSI1(GSI1PK="COURSE#c10", begins_with(GSI1SK, "STUDENT#"))

"Courses by instructor i5":
  Query GSI1(GSI1PK="INSTRUCTOR#i5", begins_with(GSI1SK, "COURSE#"))
```

[FACT] The `GSI1SK` attribute with value `"STUDENT"` and date for the first pattern is an example of
the **sparse index + static key as GSI PK** pattern. The static key (`"STUDENT"`) creates a
potential "hot partition" in the GSI — if thousands of students are being inserted per second, this
GSI partition can become a bottleneck. For high-throughput workloads, a **shard** is added to
the value: `"STUDENT#0"` to `"STUDENT#9"` randomly, and the parallel query across 10 shards is done
on the client.

### 4. When single-table design creates more problems than it solves

[OPINION — Alex DeBrie's position, The DynamoDB Book] Single-table is not for everyone, and applying it
dogmatically is a frequent mistake. The contexts where **multi-table is the safer choice**:

```
1. TEAM WITH WEAK TOOLING OR HIGH-LEVEL SDKs
   ORM-style SDKs (Java DynamoDBMapper, Java Enhanced Client, boto3 high-level Resource)
   map classes to tables. Mixing types in one table requires processing heterogeneous
   results — code becomes complex and prone to deserialization bugs.
   Symptom: "my Query code returns a mix of User and Order and I don't know which is which"

2. ENTITIES WITH INDEPENDENT LIFECYCLES AND OPERATIONS
   If you need to backup only Orders (not Users), or apply TTL only to Sessions,
   or enable InfrequentAccess storage class only for historical data — single-table forces
   the same policy for all types. Multi-table gives granular control.

3. STREAMS WITH PER-TYPE PROCESSING
   DynamoDB Streams emits an event for each modified item. In single-table, a Lambda processing
   the stream receives all types mixed and needs to filter by `entity_type`. With
   Lambda event filters by attribute this has zero invocation cost for filtered ones, but
   increases processing code complexity.

4. HIGHLY UNPREDICTABLE OR CONSTANTLY CHANGING ACCESS PATTERNS
   Single-table requires access patterns to be known and stable before design. If the
   business changes frequently and new patterns emerge, each new pattern may require a new
   GSI — and you're limited to 20 GSIs. Multi-table allows creating indexes on specific
   tables without affecting others' schemas.

5. TEAMS THAT DON'T MASTER THE DYNAMODB MENTAL MODEL
   Single-table with adjacency lists, overloaded GSIs and composite sort keys requires ALL
   developers on the team to understand the design deeply. A developer who doesn't understand and
   adds an attribute in the wrong place can silently break queries.
   Multi-table has the "strangeness" of lacking joins, but the per-table design is more intuitive.
```

[FACT] The official AWS documentation (page `bp-modeling-nosql-B`, updated in 2024) switched
to using an example of 15 access patterns implemented with **multi-table design with strategic
GSIs** as the canonical approach — representing a notable change from the
single-table evangelism of previous years. AWS itself documents the trade-offs of each
approach in `data-modeling-foundations.html` without declaring a universal winner.

[CONSENSUS] The most useful practical criterion: if you frequently need to fetch **data from multiple
entities in a single call** (e.g., user profile + their latest orders + their active sessions),
single-table with item collections is clearly superior. If entities are rarely
accessed together, multi-table is simpler to maintain without performance penalty.

---

## Practical example

**Scenario:** online course platform with many-to-many relationship between Students and
Courses, and hierarchy of Course → Module → Lesson.

### Complete schema

```
Table: edtech-platform
PK (String) + SK (String) + GSI1PK (String) + GSI1SK (String)

PK              SK                      GSI1PK          GSI1SK          Extra data
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

### Access patterns served

```
AP1: Student s1 profile
  GetItem(PK="STUDENT#s1", SK="PROFILE")

AP2: All courses student s1 is enrolled in
  Query(PK="STUDENT#s1", begins_with(SK, "ENROLLMENT#"))

AP3: All students enrolled in course c10
  Query GSI1(GSI1PK="COURSE#c10", begins_with(GSI1SK, "STUDENT#"))

AP4: All courses by instructor i5
  Query GSI1(GSI1PK="INSTRUCTOR#i5", begins_with(GSI1SK, "COURSE#"))

AP5: Complete structure of course c10 (modules + lessons)
  Query(PK="COURSE#c10", begins_with(SK, "MODULE#"))

AP6: Only modules of course c10 (without lessons)
  Query(PK="COURSE#c10", begins_with(SK, "MODULE#"))
  + FilterExpression: "attribute_not_exists(lesson_id)"  ← cost: reads everything, filters on client
  — or redesign SK for modules as "MODULE#m1#" and lessons as "MODULE#m1#L#l1"

AP7: Only lessons of module m1 of course c10
  Query(PK="COURSE#c10", begins_with(SK, "MODULE#m1#LESSON#"))
```

### Python code — adjacency list with transaction

```python
import boto3
from boto3.dynamodb.conditions import Key, Attr
from botocore.exceptions import ClientError
from datetime import datetime, timezone

dynamodb = boto3.resource("dynamodb", region_name="us-east-1")
table = dynamodb.Table("edtech-platform")


# Enroll student in course — bidirectional operation in transaction
def enroll_student(student_id: str, course_id: str) -> None:
    now = datetime.now(timezone.utc).isoformat()

    try:
        dynamodb.meta.client.transact_write_items(
            TransactItems=[
                # Edge: Student → Course (in student's partition)
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
                        "ConditionExpression": "attribute_not_exists(PK)",  # idempotent
                    }
                },
                # Inverse edge: Course → Student (in course's partition)
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
            print("Student already enrolled (condition check failed)")
        else:
            raise


# AP2: Which courses is the student enrolled in?
def get_student_courses(student_id: str) -> list[dict]:
    response = table.query(
        KeyConditionExpression=(
            Key("PK").eq(f"STUDENT#{student_id}") &
            Key("SK").begins_with("ENROLLMENT#")
        )
    )
    return response.get("Items", [])


# AP3: Which students are in the course? (via GSI1)
def get_course_students(course_id: str) -> list[dict]:
    response = table.query(
        IndexName="GSI1",
        KeyConditionExpression=(
            Key("GSI1PK").eq(f"COURSE#{course_id}") &
            Key("GSI1SK").begins_with("STUDENT#")
        )
    )
    return response.get("Items", [])


# AP5: Course structure (modules + lessons in one Query)
def get_course_structure(course_id: str) -> dict:
    response = table.query(
        KeyConditionExpression=(
            Key("PK").eq(f"COURSE#{course_id}") &
            Key("SK").begins_with("MODULE#")
        ),
        ScanIndexForward=True,  # ascending order → modules and lessons in order
    )
    items = response.get("Items", [])

    # Organize hierarchy on the client
    modules = {}
    for item in items:
        sk = item["SK"]
        parts = sk.split("#")

        if len(parts) == 2:
            # It's a module: MODULE#m1
            mod_id = parts[1]
            modules[mod_id] = {**item, "lessons": []}
        elif len(parts) == 4 and parts[2] == "LESSON":
            # It's a lesson: MODULE#m1#LESSON#l1
            mod_id = parts[1]
            lesson_id = parts[3]
            if mod_id in modules:
                modules[mod_id]["lessons"].append(item)

    return {"course_id": course_id, "modules": list(modules.values())}


# AP7: Lessons of a specific module
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

### CLI — queries with overloaded GSI

```bash
# AP3: students enrolled in course c10 (via GSI1)
aws dynamodb query \
  --table-name edtech-platform \
  --index-name GSI1 \
  --key-condition-expression "GSI1PK = :pk AND begins_with(GSI1SK, :prefix)" \
  --expression-attribute-values '{":pk":{"S":"COURSE#c10"}, ":prefix":{"S":"STUDENT#"}}'

# AP4: courses by instructor i5 (via GSI1)
aws dynamodb query \
  --table-name edtech-platform \
  --index-name GSI1 \
  --key-condition-expression "GSI1PK = :pk AND begins_with(GSI1SK, :prefix)" \
  --expression-attribute-values '{":pk":{"S":"INSTRUCTOR#i5"}, ":prefix":{"S":"COURSE#"}}'

# AP7: lessons of module m1 of course c10
aws dynamodb query \
  --table-name edtech-platform \
  --key-condition-expression "PK = :pk AND begins_with(SK, :prefix)" \
  --expression-attribute-values '{
    ":pk":{"S":"COURSE#c10"},
    ":prefix":{"S":"MODULE#m1#LESSON#"}
  }'
```

---

## Common pitfalls

### Pitfall 1: forgetting the inverse edge in adjacency list

The most common error when implementing adjacency list is writing only the edge in the direction of the
current operation. For example: when enrolling a student, the code creates `STUDENT#s1 → ENROLLMENT#c10`, but
forgets to create `COURSE#c10 → ENROLLMENT#STUDENT#s1`. Everything works until the day someone needs
AP3 ("which students are in the course?") and discovers that the course's partition doesn't have the data.
The problem is silent — no error, no exception, just empty results. Prevention is to always
implement bidirectional operations in `transact_write_items` and test both query directions
from the start of development.

### Pitfall 2: composite sort key with begins_with doesn't distinguish intermediate levels

If the hierarchical SK is `MODULE#m1#LESSON#l1`, the query `begins_with(SK, "MODULE#m1#")` returns
both the module item (`MODULE#m1`) and all its lessons (`MODULE#m1#LESSON#l1`,
`MODULE#m1#LESSON#l2`). To search for "only the module, without lessons", there's no way to express this
just with KeyConditionExpression — you would need a query with `SK = "MODULE#m1"` (GetItem
or Query with `=`), or redesign the SK so that modules and lessons have distinct prefixes that
are not prefixes of each other: e.g., modules with `MOD#m1` and lessons with `LES#m1#l1`. Planning
prefixes so there is no ambiguity between hierarchy levels is part of the design.

### Pitfall 3: GSI_PK attribute with static value at high cardinality

In the overloaded GSI, if you use `GSI1PK = "STUDENT"` to index all students and the table
has 10 million students, that GSI partition reaches the capacity of a single partition (~10 GB,
~3,000 RCU/WCU). Writes of new students will be limited to ~1,000 WCU/s on that partition — a
classic hot partition. The symptom is `ProvisionedThroughputExceededException` on the GSI even with
the main table operating normally. The solution is to add a random shard suffix:
`GSI1PK = "STUDENT#" + str(random.randint(0, 9))`, and when searching for all students make 10 parallel
queries (shards 0–9) and consolidate on the client.

---

## Reflection exercise

You are modeling a simplified professional social network. The entities are: User, Company,
and the "works at" relationship (Employment) between User and Company. A user may have worked at
multiple companies (employment history), and a company has many current and
former employees.

The access patterns are:
- Get a user's profile
- List all employment history of a user (chronological, most recent first)
- List all current employees of a company
- List former employees of a company (status = "former")
- Get the current employment of a user (only the most recent with status = "current")

Design the schema: what are the PK, SK and GSI1PK/GSI1SK values for each item type
(User, Company, bidirectional Employment). Explain how the composite sort key in the
Employment item can include the employment start timestamp to solve the "most recent
first" requirement. Identify which access pattern **is not possible** to serve with just the SK and needs
a GSI — and explain why a FilterExpression would be inadequate for this case.

---

## Resources for further study

**1. Best practices for managing many-to-many relationships in DynamoDB tables**
URL: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-adjacency-graphs.html
What you'll find: the adjacency list pattern described with the canonical example of
Invoices-Bills (many-to-many), base table and inversion GSI schema, and the
materialized graph pattern for real-time graph workflows.
Why it's the right source: primary documentation with the exact schema and motivation for the pattern.

**2. Overloading Global Secondary Indexes in DynamoDB**
URL: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-gsi-overloading.html
What you'll find: concrete example of how a single GSI with a generic `Data` attribute
serves multiple query types (by name, by warehouse, by hire date) — what
AWS calls "GSI overloading".
Why it's the right source: the only page in official documentation that names and explains the
GSI overloading pattern explicitly.

**3. Example of modeling relational data in DynamoDB**
URL: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-modeling-nosql-B.html
What you'll find: the most complete AWS example — 15 access patterns of an order
system, implemented with multi-table design and strategic GSIs. Includes the table mapping each
access pattern to the corresponding DynamoDB operation.
Why it's the right source: it is the current canonical AWS example (updated 2024) and demonstrates the
shift toward multi-table as a pragmatic approach — important for calibrating when
to use single-table vs multi-table.

**4. Data Modeling foundations in DynamoDB**
URL: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/data-modeling-foundations.html
What you'll find: explicit and objective trade-offs of single-table vs multi-table, with
a list of advantages and disadvantages of each approach (backup, encryption, streams, GraphQL, cost).
Why it's the right source: it is the most balanced official source on the single vs multi-table debate,
without dogmatism toward either approach.

**5. alexdebrie.com — DynamoDB one-to-many and single-table posts**
URLs: https://www.alexdebrie.com/posts/dynamodb-one-to-many/ and https://www.alexdebrie.com/posts/dynamodb-single-table/
What you'll find: the most cited technical posts on DynamoDB modeling outside
AWS documentation. The first covers all 1:N relationship patterns (embedding, item
collections, GSI). The second presents arguments for and against single-table with more
nuance than any re:Invent talk.
Why it's the right source: [OPINION] Alex DeBrie is the author of The DynamoDB Book and probably
the most prolific technical writer about DynamoDB outside AWS. His posts are based on
practical experience with real workloads, not evangelism of a single pattern.

---
