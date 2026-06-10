## 1. What is NoSQL?

- **NoSQL ≠ "No SQL".** Full form = **"Not Only SQL"**. Queries *can* run on it (MongoDB is an example). It means **something beyond SQL**.
- **Definition:** NoSQL databases are **non-tabular databases** that **store data differently than relational tables.** Data is the same, but it's **not stored in relational tables** (not "hobbies in one table, person in another") — it's stored together in non-tabular form.
- **Recap of what came before:** Relational Model → tables, joins, goal of **no duplicacy/redundancy** → Normalization → ACID properties. NoSQL is a *non-relational model* to store data.

### Structured vs Unstructured vs Semi-Structured Data
- **SQL requires structured data** — a **fixed schema** (e.g., Student: name, age, address, roll_no with roll_no as PK), plus **integrity constraints** (e.g., age ≥ 13 to admit).
- **NoSQL can store** structured, **unstructured**, AND **semi-structured** data:
  - **Unstructured** — text files, social media chats (each chat free-form).
  - **Semi-structured** — email (has To/From at start, but body content is unstructured; different senders → different formats).
  - **Structured** — student/employee databases with fixed schema.
- NoSQL's key requirement: **flexible schema** — you don't need all columns; some columns may be absent.

```css
DATA TYPES SPECTRUM:

  Structured           Semi-Structured         Unstructured
  ─────────────────────────────────────────────────────────►
  │                    │                       │
  Employee DB          Email                   Social media chats
  (name,age,id)        (To/From fixed,         (free-form text)
                        body free-form)

  SQL: Can handle ─────┤
  NoSQL: Can handle ───┼───────────────────────┤ (all three)
```

---

## 2. Why NoSQL Emerged (History)

- **SQL era:** Storage cost was **very high** (e.g., floppy disks of 16 KB were expensive). To minimize storage, SQL emerged: **structure the data, normalize it → no data redundancy → store minimum data**, then **retrieve via JOINs** (inner/outer joins) over foreign-key-related tables.
- **SQL is not that fast:** Data is split across tables in a fixed schema; retrieving meaningful data needs **processing (joins)**.
- **Shift:** Over time, **storage cost dropped** and **cloud systems** rose. Business need changed to: **fast access, serve many users within seconds** — redundancy is acceptable. Store data **multiple times / together** (e.g., store branch + HOD info inside student data) — "no problem, I have TB of space, I want fast access."
- **Trigger:** Cloud computing, faster internet (4G/5G), multiple servers serving many users → **NoSQL databases were born.** Storing/structuring unstructured data (emails, scientific/survey data, CCTV data for 30–40 days dumped into DB) in advance became **costly**.

```css
EVOLUTION TIMELINE:

  Past (1970s–2000s)              Present (2010s–now)
  ┌──────────────────────┐        ┌──────────────────────────┐
  │ Storage = expensive  │        │ Storage = cheap (TBs)    │
  │ Goal = minimize data │        │ Goal = fast access       │
  │ SQL = normalize data │        │ NoSQL = store together   │
  │ Use JOINs to recover │        │ No JOINs needed          │
  │ meaning              │        │                          │
  └──────────────────────┘        └──────────────────────────┘
         │                                    │
         │        Triggered by:               │
         └──────── Cloud + 4G/5G + ──────────►
                   Unstructured data explosion
```

---

## 3. Data Modeling: SQL vs NoSQL (Document model example)

**SQL (normalized):**
- `User` table: `id, first_name, last_name, cell, city` (fixed schema).
- `Hobbies` table: `user_id, hobby` — separate table because hobby is a **multi-valued attribute** (avoid redundancy).

**NoSQL (Document / JSON):**
```json
{
  "id": 1,
  "first_name": "...",
  "last_name": "...",
  "city": "...",
  "cell": "...",
  "hobbies": ["reading", "music", "coding"]
}
```
- All related data stored **together** → **no join** needed to fetch user + hobbies.
- **Downside:** Even for a small piece of info, you must **load the entire object** from DB into the application.

```css
SQL vs NoSQL — DATA MODEL COMPARISON:

  SQL (Normalized — 2 tables):
  ┌────────────────────────────┐   ┌──────────────────────┐
  │ User Table                 │   │ Hobbies Table        │
  │ id | first_name | city     │   │ user_id | hobby      │
  │  1 │ Alice      │ Delhi    │   │    1    │ reading    │
  │  2 │ Bob        │ Mumbai   │   │    1    │ music      │
  └────────────────────────────┘   │    1    │ coding     │
                                   │    2    │ gaming     │
                                   └──────────────────────┘
  To get Alice's hobbies: JOIN User + Hobbies on id=1 (extra step!)

  NoSQL (Document — 1 object):
  {                               ← Everything in one place
    "id": 1,
    "first_name": "Alice",
    "city": "Delhi",
    "hobbies": ["reading", "music", "coding"]
  }
  To get Alice's hobbies: just read this one document. Done ✅
```

---

## 4. Advantages of NoSQL

### 4.1 Flexible Schema
- No need to store everything. If an entry has no hobbies/city/cell, in **SQL you'd store NULL, NULL, NULL** (wastes space). In NoSQL you simply **omit those fields** in the JSON object → no wasted space, no predefined schema.

```
FLEXIBLE SCHEMA — NoSQL advantage:

  SQL: User with no hobbies or city
  ┌────┬────────────┬───────┬──────────┐
  │ id │ first_name │ city  │ cell     │
  │  3 │ Charlie    │ NULL  │ NULL     │  ← NULLs waste space
  └────┴────────────┴───────┴──────────┘

  NoSQL: Same user — just omit the fields
  { "id": 3, "first_name": "Charlie" }   ← clean, no NULLs, no wasted space
```

### 4.2 Horizontal Scaling (Scale-Out)

**Scaling** = upgrading the system when hardware is exhausted, data is too large, or processing is too slow — to fulfill growing user demand. Two types:

| | Vertical Scaling (Scale-Up) | Horizontal Scaling (Scale-Out) |
|---|---|---|
| **Method** | Upgrade hardware of a **single node** (more RAM, CPU, bigger disk e.g. 1TB→10TB) | **Add additional nodes** (more systems) |
| **Cost** | **Always costly** | More **cost-efficient** (many small 1GB-RAM systems) |
| **Effect** | One unit improved | **Load is shared/balanced** across nodes |
| **SQL** | Practical / possible | Difficult & impractical |
| **NoSQL** | Possible | **Very much possible** ✅ |

**Why horizontal scaling is hard in SQL (interview-critical):**
- SQL = a collection of tables. If T1, T2, T3 are spread across nodes (System1 Bangalore, System2 US, System3 Canada), and you need a **JOIN** on T1∪T2∪T3:
- Logically they're one storage, but all three must be **brought to a single node (e.g., node A4) over the network** before the join. Network transfer + join = **extremely slow.**
- Joins can't run on half the data — all parts must arrive first.
- For business use cases (open a chat, want last 2 days' messages within seconds) this is too slow → SQL not suitable.

**Why horizontal scaling is easy in NoSQL:**
- Collections are **self-contained** (all info in one JSON object), **not coupled** by relations → can be **distributed across nodes** freely because **no join is needed.**

```css
HORIZONTAL SCALING — SQL vs NoSQL:

  SQL (bad for horizontal scaling):
  ┌──────────┐   ┌──────────┐   ┌──────────┐
  │ Node 1   │   │ Node 2   │   │ Node 3   │
  │ Table T1 │   │ Table T2 │   │ Table T3 │
  └────┬─────┘   └────┬─────┘   └────┬─────┘
       │              │              │
       └──────────────┴──────────────┘
                       │
              Must JOIN all 3 tables
              → bring all data to one node
              → slow network transfer ❌

  NoSQL (good for horizontal scaling):
  ┌──────────┐   ┌──────────┐   ┌──────────┐
  │ Node 1   │   │ Node 2   │   │ Node 3   │
  │ {doc1}   │   │ {doc2}   │   │ {doc3}   │
  │ {doc4}   │   │ {doc5}   │   │ {doc6}   │
  └──────────┘   └──────────┘   └──────────┘
  Each document is self-contained.
  Query doc2? → go to Node 2 directly. No JOIN needed ✅
```

### 4.3 High Availability
- NoSQL databases do **auto-replication** across different nodes (**Replica Sets**). Data (e.g., Aadhaar data) is stored in **multiple servers**.
- If one server goes down/crashes, data is retrieved from **another server** → **reliability** is high. *(Sharding & Replica Sets enable this.)*
- "If a server fails, we can access the data from another server, as in NoSQL it's stored in multiple servers."

### 4.4 Read & Insert Operations are Easy/Fast
- *No normalization* → data **not split** across tables → **no joins** needed (joins are very costly in time complexity when data is huge).
- All related data is stored at the **same location** → fast access.
- **MongoDB rule of thumb:** *"Data that is accessed together should be stored together."*

### 4.5 Caching Mechanism
- NoSQL databases provide a **caching mechanism** — frequently accessed data can be served quickly (especially relevant for key-value stores used as caches).

### 4.6 (Trade-off) Delete & Update are Harder
- In SQL, to change hobbies you just refer to the `Hobbies` table and update.
- In NoSQL you must **load the whole object** into memory, iterate to the hobbies array, then add/delete/update → **costly in time** for update/delete; but **read is very fast.**

```css
READ vs UPDATE COST:

  READ (NoSQL wins):
  → Fetch document → all data is there. One operation. Fast ✅

  UPDATE (SQL wins):
  SQL: UPDATE Hobbies SET hobby='gaming' WHERE user_id=1  → one row, fast ✅
  NoSQL: Load entire {user: {..., hobbies: ["reading","music","coding"]}}
         → modify in memory → write whole document back → slower ❌
```

---

## 5. When to Use NoSQL (Use Cases)

1. **Fast-paced agile development** — rapid app development. (Example: developing a feature like **Reels** quickly; with SQL you must model data, avoid redundancy, design composite/multi-value tables → slower. NoSQL doesn't require worrying about duplicacy/redundancy → faster.)
2. **Storage of structured, semi-structured AND unstructured data** — all three types. (Counter-example: **banking systems** use structured data (credit cards etc.) on a single instance → SQL is fine there.)
3. **Huge volume of data** — web apps storing & quickly retrieving lots of (often unstructured) data.
4. **Requirement of scale-out** (horizontal scaling) — multiple servers. (SQL struggles because spread-out tables make joins hard.)
5. **Modern application paradigms** — microservices, real-time streaming (microservices need fast operations → fast access → NoSQL).

> For a cloud application → **NoSQL.**

```css
WHEN TO USE — QUICK DECISION GUIDE:

  Is it a cloud/web app with millions of users?        → NoSQL
  Does it need horizontal scaling?                     → NoSQL
  Is the data unstructured or semi-structured?         → NoSQL
  Is it a banking/transactional system needing ACID?   → SQL
  Is it a small app with fixed, structured data?       → SQL
  Does it need rapid development (startup, feature)?   → NoSQL
```

---

## 6. Misconceptions about NoSQL

1. **"NoSQL doesn't support ACID properties."** — Misconception. Early NoSQL DBs didn't focus on ACID, but *MongoDB* *supports* *ACID* transactions. (That's why banking systems still mostly use SQL, but it's not a hard rule.)
2. **"Relational data must use SQL / NoSQL can't store relationships."** — Misconception. NoSQL **can store relationships, just in a different way**:
   - Store related data together in one JSON object (user + hobbies), OR
   - Reference another document's `id` inside a JSON object (like a **foreign key** between two documents).
   - "Finding/modeling relationships is actually easier in NoSQL — store them together, less visualization needed."

---

## 7. Types of NoSQL Databases (Data Models)

### 7.1 Key-Value Stores
- Stores a **single key → single value** pair (no nested objects). Key can be string/int; value can be JSON, binary object, string, array — anything. Acts like a **dictionary**.
```json
{ "id": "lakshay" }   // just key → value, nothing else
```
- **Use case:** Shopping carts (hash key → cart value), simple paired storage; also **user preferences** and **user profiles**.
- Key-value stores use **compact, efficient index structures** to locate a value by its key quickly and reliably — ideal for systems needing data retrieval in **constant time**.
- **Optimal use-cases for a key-value approach:**
  - **Real-time random data access** — e.g., user session attributes in an online application such as gaming or finance.
  - **Caching mechanism** for frequently accessed data or configuration based on keys.
  - Applications designed on **simple key-based queries**.
- **Examples:** **Oracle NoSQL**, **Redis**, **Amazon DynamoDB**. MongoDB also supports it.

### 7.2 Column-Oriented (Columnar / C-Store / Wide-Column)
- **SQL stores data row-wise** (row-optimized): all of a row's data stored together because you usually want the whole row. e.g. memory: `Mat, Delhi, 27, Rd, Jaipur, 27, ...`
- **Columnar stores data column-wise:** all names together, all cities together, all ages together. Adding a new tuple inserts into the *middle* of each column → **insert is slow**, but **read is fast**.
- **Benefit — Analytics / Aggregation:** To compute `AVG(age)`:
  - **Row-wise:** must jump in steps of 3 (skip name, city → pick age) → slow disk access.
  - **Column-wise:** all ages stored contiguously → read them in one bunch → **fast**.
- **Use case:** Heavy **analytics/aggregation** (e.g., industrial machine data, average age, peak time, repeated aggregate functions). Read fast; insert slow.
- **Examples:** **Cassandra** (a good industry example), **RedShift**, **Snowflake**.

```css
ROW-WISE vs COLUMN-WISE STORAGE:

  Data: (name, city, age)
  Row 1: Alice, Delhi, 25
  Row 2: Bob,   Mumbai, 30
  Row 3: Carol, Pune, 27

  ROW-WISE (SQL):              COLUMN-WISE (Columnar NoSQL):
  [Alice,Delhi,25]             [Alice, Bob, Carol]   ← all names together
  [Bob,Mumbai,30]              [Delhi, Mumbai, Pune] ← all cities together
  [Carol,Pune,27]              [25, 30, 27]          ← all ages together

  To compute AVG(age):
  Row-wise:   Alice,Delhi,→25→ Bob,Mumbai,→30→ Carol,Pune,→27  (skip,skip,read)
  Column-wise: [25, 30, 27] → read all at once! Much faster ✅
```

### 7.3 Document-Based
- Stores **documents similar to JSON objects.** Each document = a JSON object with **multiple fields and values** (vs key-value's single pair). Values can be string, number, boolean, array, object — anything.
- Documents can **reference each other** (e.g., `user` document id = 1 referenced in a `contact`/`access` document with id = 1 → acts like a **foreign key** relation).
- **Use case:** E-commerce / trading platforms; **general-purpose** (handles most use cases). **Supports ACID transactions.**
- **Examples:** **MongoDB** (prime example, most-used in cloud) and **CouchDB**.

### 7.4 Graph-Based Stores
- Data viewed as **nodes (vertices)** and **edges (relations)**. Optimized to store and search **relationships** directly.
- **Prime example:** **Facebook friends / friends-of-friends.** Store profiles (A, B, C, D, E) as nodes; store edges (A–B friends, B–C friends, etc.) as relations.
- **Why:** "Connections are **first-class elements** of the database, **stored directly**" — overcomes the **overhead of joining multiple tables**. In SQL you'd store FK relations in a separate table and JOIN; graph DBs avoid this overhead.
- **Use cases:** Social networking, knowledge graphs, **fraud detection** (card used here, then this ATM, then that ATM → build connections).
- **Note:** Very few real-world business systems can survive **solely** on graph queries → graph databases are usually run **alongside other, more traditional databases.**
- **Example:** **Neo4j.**

> Across all types, the difference is mainly *how data/connections are stored*: key-value (single pair), columnar (analytics), document (JSON fields), graph (direct connections).

```css
4 TYPES OF NoSQL — VISUAL SUMMARY:

  1. KEY-VALUE (Redis):          2. COLUMNAR (Cassandra):
  ┌──────────┬─────────────┐     col:  Name    City    Age
  │ "user:1" │ "Alice"     │     ─────────────────────────
  │ "cart:5" │ [{item:...}]│     data: Alice   Delhi   25
  └──────────┴─────────────┘           Bob     Mumbai  30
  Dictionary-like, simple pairs.  Analytics-optimized storage.

  3. DOCUMENT (MongoDB):         4. GRAPH (Neo4j):
  {                               (A)───friends───(B)
    "name": "Alice",                │                │
    "hobbies": ["coding"],        knows            knows
    "address": {...}                │                │
  }                               (C)─────────────(D)
  Rich nested JSON objects.       Nodes + Edges for relationships.
```

---

## 8. Disadvantages of NoSQL

1. **Data Redundancy → more storage.** Same data in same use case takes **more memory/storage** than SQL (SQL minimizes redundancy via normalization). Larger DB size. *(Minor drawback today since storage is cheap; trade storage for fast access.)* Some NoSQL databases also support **compression** to reduce the storage footprint.
2. **Update & Delete are costly** — related data stored together; must load whole object.
3. **No single NoSQL data model fulfills all application needs** — key-value vs document vs columnar vs graph each serve different needs (e.g., graph excels at relationships but not general-purpose). SQL mostly fulfills all app needs (except cloud scalability). *(MongoDB cleverly leverages its document model to also act key-value-like.)*
4. **ACID not generally supported** across all NoSQL (MongoDB is an exception).
5. **Weak consistency constraints in the DB itself** — most NoSQL databases do **not** enforce strong consistency constraints inside the DB engine the way SQL does (e.g., CHECK constraints, NOT NULL per column). You typically add validation in your application layer before writing. Some databases like MongoDB do offer schema validation rules, but this is opt-in and less strict than SQL's declarative constraints.

---

## 9. SQL vs NoSQL — Comparison Table (the main interview answer)

| Aspect                  | SQL                                                                                                                                     | NoSQL                                                                                                                       |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| **Storage Model**       | Tables: fixed rows, fixed columns (missing entry → NULL)                                                                                | JSON documents / key-value / columnar / nodes-edges (graph); dynamic columns                                                |
| **Schema**              | **Fixed** schema                                                                                                                        | **Flexible** schema                                                                                                         |
| **Scaling**             | Primarily **vertical**; horizontal is possible but difficult because JOINs across shards require expensive cross-node network transfers | Primarily **horizontal** (scale-out); self-contained documents/records distribute naturally with no cross-node JOINs needed |
| **ACID**                | **Supported** (primary goal)                                                                                                            | Generally **not** supported (MongoDB: supported)                                                                            |
| **Joins**               | **Needed** to get meaningful data from related tables                                                                                   | **Not needed** — related data stored together                                                                               |
| **Development History** | Storage was costly → minimize duplicacy/redundancy                                                                                      | Focus on **scaling + fast access + rapid application change (agile)**                                                       |
| **Data→Object Mapping** | **Required** (manually map DB rows to app objects; Java **DAO pattern** — Data Access Object)                                           | Many do not require ORMs — MongoDB documents map **directly** to data structures in language (e.g., Python class)           |
| **Examples**            | Oracle, MySQL, Microsoft SQL Server, PostgreSQL                                                                                         | Document: MongoDB, CouchDB; Key-value: Redis, DynamoDB; Wide-column: Cassandra, HBase; Graph: Neo4j, Amazon Neptune         |
| **Primary Use**         | General-purpose                                                                                                                         | Document: general; Key-value: large data store; Wide-column: fast querying/analytics; Graph: retrieve relationships         |

### Data → Object Mapping (detail)
- **SQL:** App (e.g., front-end `Student` class) retrieves rows from DB server; you must **explicitly map** each field: `student.name = serverStudentName`, etc. → use the **DAO (Data Access Object) pattern** in Java.
- **NoSQL (MongoDB):** Documents **map directly** to language data structures (e.g., a Python `Student` class) → `student = <whatever came from server>` auto-spreads; **no explicit field-by-field mapping** needed.

```css
DATA → OBJECT MAPPING:

  SQL (requires DAO pattern):
  DB Row: [id=1, name="Alice", age=25]
  Code:   student.id   = row[0]     ← manual field-by-field mapping
          student.name = row[1]
          student.age  = row[2]

  NoSQL (direct mapping):
  DB Doc: {"id": 1, "name": "Alice", "age": 25}
  Code:   student = db.find(id=1)   ← document maps directly to object ✅
          student.name  → "Alice"   ← no manual mapping needed
```

---
## DAO (Data Access Object)

A **DAO** is a design pattern that provides an abstraction layer between the application and the database.

- Handles all database operations (CRUD: Create, Read, Update, Delete).
- Hides SQL queries and database details from business logic.
- Improves code maintainability, reusability, and modularity.
- Makes it easier to switch databases without changing application logic.

**Flow:**  
Application → DAO → Database

**Example:**  
`StudentDAO.getStudent(id)` retrieves student data without the application directly writing SQL queries.

---
## Key Takeaways
- **NoSQL = "Not Only SQL"** — non-tabular, flexible schema, stores structured/semi/unstructured data.
- Emerged due to **cheap storage + cloud + need for fast, scalable access.**
- Big advantage: **horizontal scaling** (self-contained collections, no joins) + **high availability** (replica sets).
- Trade-off: **fast reads** but **costly writes/updates** and **more storage** (redundancy).
- **4 types:** Key-Value, Columnar, Document, Graph.
- **MongoDB** is the most important NoSQL DB (general-purpose, supports ACID).
- The interview answer is the **SQL vs NoSQL table** + **when to use NoSQL** list.
