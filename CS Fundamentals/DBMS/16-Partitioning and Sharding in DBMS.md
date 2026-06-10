## 1. The Core Problem (Why we need DB Optimization)

When you build a system, DBMS makes storing data easy. But two problems arise as the system grows:

1. **Huge Data** → **Manageability issue**. So much data that a single node cannot manage it well.
2. **Large Number of Requests** → A single system cannot cater to millions of requests efficiently.

When these two problems occur, we **distribute the data**, which gives us a **Distributed Database**.

To solve this, we **apply Database Optimization Techniques**, which:
- Make data **easily manageable** (because data is huge).
- **Reduce the response time** for the large number of incoming requests.

```css
        Small System (Works Fine)           Large System (Problems Arise)
        ┌──────────────────────┐            ┌──────────────────────────────┐
        │   Single DB Server   │            │      Single DB Server        │
        │   ┌──────────────┐   │            │   ┌──────────────────────┐   │
        │   │  100 rows    │   │            │   │  100 MILLION rows    │   │
        │   │  10 users    │   │   ───────► │   │  10 MILLION users    │   │
        │   │  works fine  │   │  (grows)   │   │  ← Slow! Hangs!      │   │
        │   └──────────────┘   │            │   └──────────────────────┘   │
        └──────────────────────┘            └──────────────────────────────┘
                                                 Problem 1: Data too huge
                                                 Problem 2: Too many requests
```

---

## 2. Approach 1 — Scale Up (Vertical Scaling)

- **Scale Up** = increase the hardware (increase HDD, CPU, RAM) of a single node.
- Example: a system has 1 TB HDD, some CPU (XYZ), some RAM (X). When data and requests double (2x data, 2n requests), the instinct is to double everything → 2 TB, 2x CPU, 2x RAM.

```css
        Before Scale-Up                  After Scale-Up
        ┌────────────────┐               ┌────────────────┐
        │  Server Node   │               │  Server Node   │
        │  CPU:  1x      │    Upgrade    │  CPU:  2x      │
        │  RAM:  8 GB    │  ──────────►  │  RAM:  16 GB   │
        │  HDD:  1 TB    │  (expensive!) │  HDD:  2 TB    │
        └────────────────┘               └────────────────┘
         Handles N requests               Handles ~1.5N requests
                                          (NOT 2N — not linear!)
```

**Problems with Scale Up:**
1. **Response time does NOT become half** even if you double everything (practically observed — it doesn't scale linearly).
2. **Cost increases a lot** — increasing TB, CPU, RAM is *expensive*.
- Therefore, scaling up is **not very logical in terms of money**. Ultimately everything comes down to money, and you want minimum cost.

---
## 3. Approach 2 — Clustering (Replica Sets)

- Clustering (covered last lecture) = make *copies* **of the same database instance**.
- All data is **replicated** → redundancy is included.

```css
        ┌──────────────────────────────────────────────────────┐
        │                  Clustering Setup                    │
        │                                                      │
        │   ┌──────────────┐       ┌──────────────────────┐   │
        │   │   MASTER     │──────►│  Replica 1 (Copy)    │   │
        │   │  (Original)  │  rep  │  Same data as Master │   │
        │   │              │──────►│  Replica 2 (Copy)    │   │
        │   │  All WRITES  │  lag! │  (slightly behind)   │   │
        │   └──────────────┘       └──────────────────────┘   │
        │                                                      │
        │   ⚠ Problem: Propagation delay → replica may lag    │
        └──────────────────────────────────────────────────────┘
```

**Problem with Clustering:**
- There is a **Master node** and **Replicas**.
- All **updates always come on the Master**.
- The Master **propagates** these updates to the replicas → propagation causes **delay**.
- This delay can cause **eventual** inconsistency: one replica may be updated while another is not yet updated.

Clustering is a good method, but **one level above** there is a better method → **Partitioning**.

---

## 4. Partitioning

- Partitioning is a **Scale Out** (Horizontal Scaling) technique.
- We do NOT increase a single node's CPU/capacity. Instead, we **partition the data itself** and **add new nodes**.
- (Clustering is also horizontal scaling — you add nodes; but there you copy data, here you divide data.)

```css
        Clustering (Copy data)            Partitioning (Divide data)
        ┌────────┐  ┌────────┐            ┌────────┐  ┌────────┐
        │Server 1│  │Server 2│            │Server 1│  │Server 2│
        │ALL data│  │ALL data│            │Part A  │  │Part B  │
        │(copy)  │  │(copy)  │            │rows1-N │  │rows N+1│
        └────────┘  └────────┘            └────────┘  └────────┘
          Same data everywhere              Different data on each node
```

**How it cuts the problem down:** Partitioning divides a big database containing **data metrics and indexes** into smaller, handy slices of data called **partitions**. The **partitioned tables are used directly by SQL queries without any alteration**. Once the database is partitioned, the **Data Definition Language (DDL)** can easily work on the smaller partitioned slices instead of handling the giant database altogether — this is how partitioning cuts down the problems in managing large database tables.

**Definition:**
> *"Partitioning is the technique used to divide stored database objects into separate servers. Due to this, there is an increase in performance and controllability."*

- When we **horizontally scale** our machines/servers, it gives a **challenging time dealing with relational databases** because it is quite tough to maintain the relations. But if we apply partitioning to a database that is **already scaled out** (equipped with multiple servers), we can **partition our database among those servers** and handle the big data easily.

**When to apply (very important):** We apply these techniques only when we want to **reduce response time**. Response time increased because data is huge — traversing huge data takes a lot of time.

### Example Table — Student
Columns: `Name`, `ID`, `Class`, `Address`, `Phone Number` (with many students).

### Two Types of Partitioning

#### 4.1 Vertical Partitioning
- Divide the table **column-wise**.
- Some columns stored in server S1, some in S2, some in S3.
- **Need to access different servers to get complete tuple information.**
- Example: first 2 columns in S1, next 2 columns in S2. To get a complete tuple, you must **access both servers** (some data from S1, some from S2).

```css
        Original Student Table
        ┌────────┬────────┬───────┬──────────┬──────────────┐
        │  Name  │   ID   │ Class │ Address  │ Phone Number │
        ├────────┼────────┼───────┼──────────┼──────────────┤
        │  Alice │  101   │  10   │  Delhi   │  9999900001  │
        │  Bob   │  102   │  11   │  Mumbai  │  9999900002  │
        └────────┴────────┴───────┴──────────┴──────────────┘

        After VERTICAL Partitioning (column-wise split):

        Server S1                        Server S2
        ┌────────┬────────┐              ┌───────┬──────────┬──────────────┐
        │  Name  │   ID   │              │ Class │ Address  │ Phone Number │
        ├────────┼────────┤              ├───────┼──────────┼──────────────┤
        │  Alice │  101   │              │  10   │  Delhi   │  9999900001  │
        │  Bob   │  102   │              │  11   │  Mumbai  │  9999900002  │
        └────────┴────────┘              └───────┴──────────┴──────────────┘
        ⚠ To get full row, you MUST query BOTH servers and JOIN the results.
```

#### 4.2 Horizontal Partitioning
- Divide the table **tuple-wise (row-wise)**.
- Example: IDs/roll numbers 1 to 25k stored in S1; 25k+1 to 100k stored in S2.
- This gives **independent chunks of data tuples stored in different servers**.
- Example: if you need students 1–10000, you fetch only from S1; S1 is **independent of** S2 (you don't access S2).

```css
        Original Student Table (100k rows)
        ┌────────┬────────┬───────┬──────────┐
        │  Name  │   ID   │ Class │ Address  │
        ├────────┼────────┼───────┼──────────┤
        │  Alice │   1    │  10   │  Delhi   │
        │  Bob   │   2    │  11   │  Mumbai  │
        │   ...  │  ...   │  ...  │   ...    │  ← 100,000 rows total
        └────────┴────────┴───────┴──────────┘

        After HORIZONTAL Partitioning (row-wise split):

        Server S1 (rows 1 – 25,000)      Server S2 (rows 25,001 – 100,000)
        ┌────────┬────────┬───────┐       ┌────────┬────────┬───────┐
        │  Name  │   ID   │ Class │       │  Name  │   ID   │ Class │
        ├────────┼────────┼───────┤       ├────────┼────────┼───────┤
        │  Alice │   1    │  10   │       │  Mark  │ 25001  │  9    │
        │  Bob   │   2    │  11   │       │  Sara  │ 25002  │  10   │
        │  ...   │  ...   │  ...  │       │  ...   │  ...   │  ...  │
        └────────┴────────┴───────┘       └────────┴────────┴───────┘
        ✓ Query for ID=500? → go ONLY to S1. S2 is not touched. Faster!
```

**Definition of Partitioning (in words):** Simply dividing your data/table — storing some data on one node and some data on another node. Divide a big problem into small parts and solve.

---

## 5. When is Partitioning Applied?

1. **When data becomes huge** → managing and dealing with it becomes a tedious task.
2. (Even more important reason) **When the number of requests becomes so high** that a **single database server is not able to serve your requests in stipulated time**.

In these cases, we partition things horizontally or vertically.

---

## 6. Advantages of Partitioning (5 Advantages)

1. **Parallelism**
   - Example: chunk 1 in S1, chunk 2 in S2. Multiple requests come → you **filter out** the requests.
   - Requests for IDs 1–10000 → sent to S1; rest → sent to S2.
   - This establishes a **kind of parallelism**.

2. **Availability Increases**
   - If node S1 crashes, the requests corresponding to the other node can still be served. (All DB optimization techniques increase availability.)

3. **Performance Increases**
   - System response time reduces because each node (S1, S2) is now **less loaded**.
   - If all info were in one system, it would be a heavily-loaded system taking more time. Also, requests get divided → better performance is guaranteed.

4. **Manageability Increases**
   - A DBA who needs to change students 1–10000 just goes to S1 and changes it (faster).
   - You get a **bird's-eye view**: half data is here, half is there → work where the relevant data is.

5. **Reduced Cost**
   - Instead of vertical scaling (expensive — increasing CPU/HDD/RAM of one node), you can bring in **another smaller-capacity CPU/node** because requests get divided across nodes.

```css
        Without Partitioning               With Partitioning
        ┌──────────────────┐               ┌───────────┐  ┌───────────┐
        │  ONE Big Server  │               │  Server 1 │  │  Server 2 │
        │  ← All requests  │               │ ← req 1-N │  │ ← req N+1 │
        │  CPU:  100%      │               │ CPU: ~50% │  │ CPU: ~50% │
        │  Slow, expensive │               │  fast!    │  │  fast!    │
        └──────────────────┘               └───────────┘  └───────────┘
                                           If S1 crashes → S2 still serves!
                                           (Availability ↑)
```

---

## 7. Distributed Database (Result of these techniques)

After partitioning, you divided your database into different servers — some data here, some there — **but logically it is still the same database** (e.g. still the Student table / University database).

**Definition:**
> A **Distributed Database** is a **single logical database that is spread across multiple locations / servers, logically interconnected together by a network.**

```css
        ┌─────────────────────────────────────────────────────────────┐
        │               University Database (Logical)                 │
        │  ← This is ONE database in the user's/application's view    │
        │                                                             │
        │   ┌──────────────┐    Network    ┌──────────────┐          │
        │   │  Server S1   │◄─────────────►│  Server S2   │          │
        │   │  Rows 1–25k  │               │  Rows 25k+1  │          │
        │   └──────────────┘               └──────────────┘          │
        │           Physically separate, Logically ONE                │
        └─────────────────────────────────────────────────────────────┘
```

Whenever you apply **Clustering, Partitioning, and Sharding**, you ultimately **distribute the database**.
**Need:** Apply these when requests increased and data became huge.

---

## 8. Sharding

- Sharding is a **technique to implement Horizontal Partitioning** — an **extension** of (horizontal) partitioning.

**Fundamental idea:**
> *"Sharding is the idea that instead of having all the data set on one instance, we split it up and introduce a routing layer so that we can forward the request to the right instance that actually contains the data."*

```css
        Without Sharding (Horizontal Partition only)
        ┌─────────────────────────────────────────────────┐
        │  Application / Client                           │
        │  → Sends request to DB... but which one?        │
        │    No routing logic! Developer must handle it.  │
        └─────────────────────────────────────────────────┘

        With Sharding (Horizontal Partition + Routing Layer)
        ┌──────────────────────────────────────────────────────────┐
        │                     Client Request                       │
        │                   "Get roll_no = 5000"                   │
        └──────────────────────────┬───────────────────────────────┘
                                   │
                                   ▼
        ┌──────────────────────────────────────────────────────────┐
        │               ROUTING LAYER (Lookup Layer)               │
        │  Shard Key = roll_no                                     │
        │  Rule: roll_no 1–10k   → Shard 1                        │
        │        roll_no 10k+1.. → Shard 2                        │
        │  roll_no=5000 → goes to Shard 1                         │
        └──────────────┬───────────────────────────────────────────┘
                       │
               ┌───────▼──────┐           ┌──────────────┐
               │   SHARD 1    │           │   SHARD 2    │
               │  roll 1–10k  │           │  roll 10k+1+ │
               └──────────────┘           └──────────────┘
```

### Example — Student Data
- Roll numbers 1–10k stored in S1; 10k+1–100k stored in S2 (Horizontal Partition → tuple-wise division; whole tuples placed in respective nodes).
- S1 and S2 are **independent**: requests for 1–10k go to S1, rest go to S2.

### The Routing Layer (key requirement of Sharding)
- The split nodes are called **Shards** (Shard 1, Shard 2).
- Shard 1: roll numbers 1–10k. Shard 2: roll numbers >10k–100k.
- When a request comes, you must write a **Routing Layer** (in your application/software layer) — also called a **lookup / intermediate layer**.
- This layer decides **which shard the request should go to** (S1 DB instance vs S2 DB instance).
- This is an **additional capability / additional implementation** you must build.

### Example — Invoice Table
- Rows split into Shard 1 and Shard 2 (Database Shard 1, Database Shard 2) via Horizontal Partitioning.
- The **Shard Key (Partition Key)** here = **Customer ID** (the key on which you partition).
- The **Primary Key** is `Invoice ID`, but the **Shard Key can be something different** (here Customer ID).

```css
                    LOGICAL TABLE: Invoices
┌────────────┬─────────────┬────────┐
│ Invoice ID │ Customer ID │ Amount │
├────────────┼─────────────┼────────┤
│ INV001     │ C01         │ 500    │
│ INV002     │ C75         │ 200    │
│ INV003     │ C02         │ 800    │
│ INV004     │ C88         │ 350    │
└────────────┴─────────────┴────────┘
Primary Key = Invoice ID  (uniquely identifies a row)
Shard Key   = Customer ID (decides which shard stores the row)

                 Sharding Logic
        Route using Customer ID range

                Customer ID
                      │
          ┌───────────┴───────────┐
          │                       │
      C01 – C50              C51 – C100
          │                       │
          ▼                       ▼

      SHARD 1                  SHARD 2
┌────────────┐           ┌────────────┐
│ INV001 C01 │           │ INV002 C75 │
│ INV003 C02 │           │ INV004 C88 │
└────────────┘           └────────────┘

Key Point:
Primary Key identifies the record.
Shard Key decides where the record is stored.
They may be the same or different.
```

The server in sharding is just called a Shard.

---

## 9. Partitioning vs Sharding (Important Clarification)

- In **System Design interviews / industry**, people use **Partitioning and Sharding as synonyms** (e.g. "vertical sharding", "horizontal sharding").
- Strictly, they are **not** exact synonyms, but normally used interchangeably in industry.
- **By theory:** **Sharding is a way of doing Horizontal Partitioning.**

```css
        ┌──────────────────────────────────────────────┐
        │              Partitioning                    │
        │  ┌─────────────────┐  ┌─────────────────┐   │
        │  │ Vertical Part.  │  │ Horizontal Part. │   │
        │  │ (column-wise)   │  │  (row-wise)      │   │
        │  └─────────────────┘  └────────┬────────┘   │
        │                                │             │
        │                         ┌──────▼──────┐      │
        │                         │  SHARDING   │      │
        │                         │ (adds the   │      │
        │                         │  Routing    │      │
        │                         │   Layer)    │      │
        │                         └─────────────┘      │
        └──────────────────────────────────────────────┘
        Sharding ⊂ Horizontal Partitioning
        In interviews: often used interchangeably — that's fine!
```

---

## 10. Pros & Cons of Sharding

### Pros
- Parallelism increases
- Availability increases
- Manageability increases
- Cost reduces
- It is a technique to **increase Scalability** (Scale Out) — you avoid costly vertical scaling.

### Cons

1. **Complexity — Mapping / Routing Layer**
   - You need a mapping to identify which shard a particular request belongs to (based on the **Partition Key**, e.g. roll number for Student, Customer ID for Invoice).
   - You must write an **additional Routing Layer** yourself.

2. **Non-Uniformity**
   - Ideally you want a **uniform division** (e.g. 6 rows → 3 here, 3 there).
   - But since the Shard Key (e.g. Customer ID) is not the primary key, one customer (Customer ID 2) might have placed 10,000 orders → that portion grows → **information becomes more on one side, less on the other** → non-uniform.
   - **Solution:** **Re-shard** the table/system from time to time — change the partition key / re-shard based on the partition key. **Re-sharding is necessary.**

```css
        Ideal (Uniform)                   Reality (Non-Uniform — Hot Shard)
        ┌──────────┐  ┌──────────┐        ┌──────────────────┐  ┌───────┐
        │  Shard 1 │  │  Shard 2 │        │     Shard 1      │  │Shard 2│
        │  3 rows  │  │  3 rows  │        │  10,000+ rows    │  │ 2 rows│
        │  ████    │  │  ████    │        │  ████████████    │  │ █     │
        └──────────┘  └──────────┘        └──────────────────┘  └───────┘
        Balanced ✓                         Overloaded! → Re-shard needed
```

3. **Not well suited for Analytical type of queries** (the **Scatter-Gather problem**)
   - Because the data is spread across different DB instances, an analytical query must **scatter** to all instances and **gather** the partial results back — this is known as the **Scatter-Gather problem**.
   - Example: a new column `Amount`. You want total amount each customer ordered (aggregation).
   - On a **single node**, you'd simply run an **aggregation function** and SQL returns the sigma/answer from one node.
   - With sharding, data is spread across multiple nodes → SQL/DBMS (or you) must: go to node 1 → get its sum, go to node 2 → get its sum, then **add both sums** → return answer. This introduces complexity.

```css
        Single Node (Simple Aggregation)
        ┌──────────────────┐
        │  SELECT SUM(amt) │  → one query, one answer ✓
        │  FROM invoices;  │
        └──────────────────┘

        Sharded (Complex Aggregation)
        ┌──────────────────────────────────────────────────┐
        │ 1. Query Shard 1 → SUM = 50,000                 │
        │ 2. Query Shard 2 → SUM = 30,000                 │
        │ 3. Application layer: 50,000 + 30,000 = 80,000  │
        │                          ↑ You must handle this! │
        └──────────────────────────────────────────────────┘
```

   - Pros come with cons.

---

## 11. Takeaways

- Handling N requests on a single system / reducing response time:
  - **Shard it** → divide across different nodes → write a routing layer that sends each request to its particular shard.
  - Result: manageability ↑, system scales out, scalability ↑, availability ↑, parallelism ↑, performance ↑, and you **avoid costly vertical scaling**.
- The three DB Optimization techniques — **Clustering (Replica Sets), Partitioning, Sharding** — share the same **core idea** of distributing the database.
- Sharding also has SQL queries / **DDL (Data Definition Language)**.

---

## Quick Revision

- Problems: **huge data** (manageability) + **too many requests** (response time) → distribute → Distributed DB.
- **Scale Up** (vertical): increase hardware → costly, non-linear → avoid.
- **Clustering**: copy DB; Master takes updates, propagates to replicas → propagation delay.
- **Partitioning** (Scale Out / Horizontal Scaling): divide data, add nodes.
  - **Vertical** = column-wise (need multiple servers for full tuple).
  - **Horizontal** = row/tuple-wise (independent chunks).
- **5 Pros**: Parallelism, Availability, Performance, Manageability, Reduced Cost.
- **Sharding** = technique to implement Horizontal Partitioning + **Routing Layer**.
  - **Shard Key / Partition Key** decides routing (can differ from primary key).
  - **Cons**: routing-layer complexity, **non-uniformity** (fix via **re-sharding**), **bad for analytical/aggregation queries**.
- Industry: Partitioning ≈ Sharding (synonyms); by theory Sharding = horizontal partitioning.
