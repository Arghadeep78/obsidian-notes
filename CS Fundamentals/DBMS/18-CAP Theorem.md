# 18 — CAP Theorem

> Relates to **Distributed Storage** — used when building cloud systems / large distributed databases.

---

## 1. Context — Why CAP Theorem?

- We scale up only to a level, then **scale out** (Horizontal Scaling) → make multiple nodes / replicas.
- When replicas are made / a cloud system / distributed storage is built, one theorem is very useful **to design efficient distributed storages** → the **CAP Theorem**.

```
        Single Node             Scale Out (Distributed)
        ┌──────────┐            ┌──────────┐   ┌──────────┐   ┌──────────┐
        │  DB      │  grows →   │  Node A  │   │  Node B  │   │  Node C  │
        │  (all    │            │ (replica)│   │ (replica)│   │ (replica)│
        │  data)   │            └──────────┘   └──────────┘   └──────────┘
        └──────────┘                    Now: multiple nodes → need to decide
                                        What guarantees can we provide?
                                        → CAP Theorem answers this
```

---

## 2. The Three Properties (C, A, P)

### 2.1 Consistency (C)
- When the database is spread over multiple nodes (replicas), the **values of data on all nodes should be the same**.
- If value of `x` is 10 on Node A, it should be 10 on Node B too. **Data across the nodes must be consistent.**
- **How:** When data is written on a single node, *it should be replicated / broadcasted to other nodes in the system.*

```
        Consistent State:
        ┌──────────┐    ┌──────────┐    ┌──────────┐
        │  Node A  │    │  Node B  │    │  Node C  │
        │  x = 10  │    │  x = 10  │    │  x = 10  │
        └──────────┘    └──────────┘    └──────────┘
        All nodes agree: x = 10  ✓ CONSISTENT

        Inconsistent State (Bad):
        ┌──────────┐    ┌──────────┐    ┌──────────┐
        │  Node A  │    │  Node B  │    │  Node C  │
        │  x = 20  │    │  x = 10  │    │  x = 15  │
        └──────────┘    └──────────┘    └──────────┘
        Nodes disagree! ✗ NOT CONSISTENT
```

### 2.2 Availability (A)
- In a distributed system / distributed database, availability means *the system remains operational all of the time.*
- If there are multiple nodes and some nodes fail, the system should still be **available** (some latency/delay is OK, but the system must be available).
- *Every request will get a response regardless of the individual state of the node.*
- Unlike a consistent system, there is **no guarantee that the response will be the most recent write operation.**

```
        High Availability:
        ┌──────────┐    ┌──────────┐    ┌──────────┐
        │  Node A  │    │  Node B  │    │  Node C  │
        │  ALIVE   │    │  DOWN ⚠  │    │  ALIVE   │
        └──────────┘    └──────────┘    └──────────┘
                │                              │
                └──────────────────────────────┘
                     System still responds ✓
                     (Node B is down, but A and C serve requests)
                     → AVAILABLE
```

### 2.3 Partition Tolerance (P)
- A **Partition** = a **break in communication between the nodes**.
  - Setup: Node A (Primary) and Node B (Replica) need a communication link to replicate. If that **communication link breaks → that is a Partition**.
- **Partition Tolerance** = if a partition occurs, the system **should be tolerant of it** — i.e. the system should **not fail**, it stays up.
  - *"Whenever a distributed system encounters a partition... it means there is a break in communication between the nodes."*
  - There may be **delays in messages / inconsistency**, but the system stays up.
- *"To have partition tolerance, the system must replicate records across combinations of nodes and network"* — when the node comes back up and communication is re-established, you must replicate.

```
        What is a Network Partition?

        Normal (No Partition):
        ┌──────────┐  ←────────────────►  ┌──────────┐
        │  Node A  │   communication OK   │  Node B  │
        │ (Primary)│                      │(Replica) │
        └──────────┘                      └──────────┘

        Partition Occurs (Communication Link Breaks):
        ┌──────────┐   ✗ link broken ✗   ┌──────────┐
        │  Node A  │ ─ ─ ─ ─ ─ ─ ─ ─ ─  │  Node B  │
        │ (Primary)│   cannot replicate   │(Replica) │
        └──────────┘                      └──────────┘
        This break is called a PARTITION.

        Partition Tolerant = System keeps running despite this break.
```

---

## 3. The CAP Theorem Statement

> **Consistency, Availability, and Partition Tolerance — all three cannot be achieved simultaneously.**
> Whenever you build a distributed system, **only two of these three** properties can simultaneously exist.

```
        The CAP Triangle — Pick any 2, sacrifice the 3rd

                           Consistency (C)
                                 /\
                                /  \
                    CP side →  /    \  ← CA side
                              / (all \
                             /  three \
                            /  cannot  \
                           /  coexist   \
                          ──────────────
               Availability (A)          Partition Tolerance (P)
                              ↑
                             AP side

        Each EDGE of the triangle = a valid pair you can guarantee.
        The CENTER = impossible to achieve all three simultaneously.

        CA edge → give up P  (only works on single node; no distribution)
        CP edge → give up A  (turn off inconsistent nodes during partition)
        AP edge → give up C  (accept stale reads during partition)
```

**Hypothetical exception (all three together):**
- If you have a **single node** (a small application on one computer), then a **partition can never happen** (you never divided into multiple nodes).
- So for a **single-node (non-distributed) database**, all three can hold. But if it is **distributed**, only **two of the three** can be established at a time.

**Other wording (the trade-off):**
> The theorem is a **trade-off between Consistency and Availability when there is a Partition.**
> If a partition (communication break) occurs, you must choose **either Consistency or Availability.**

---

## 4. Proof / Intuition (Example)

Setup: a single DB grows large → scaled out → divided into **Node A (Primary — handles all WRITE operations)** and **Node B (Secondary/Replica — handles READ operations)**. Now a **partition occurs** (communication break).

```
        Setup Before Partition:
        ┌──────────────┐  replication  ┌──────────────┐
        │   Node A     │ ────────────► │   Node B     │
        │  (Primary)   │               │  (Secondary) │
        │  x = 10      │               │  x = 10      │
        │  ALL WRITEs  │               │  ALL READs   │
        └──────────────┘               └──────────────┘

        Partition Occurs:
        ┌──────────────┐  ✗ BROKEN ✗  ┌──────────────┐
        │   Node A     │ ─ ─ ─ ─ ─ ─  │   Node B     │
        │  (Primary)   │   no sync     │  (Secondary) │
        │  x = 10      │               │  x = 10      │
        └──────────────┘               └──────────────┘
        A WRITE comes: x = 20 → goes to Node A (the only node accepting writes)
        Node A updates: x = 20 (x changed from 10 to 20)
        But cannot replicate to Node B (link is broken!)
        Node B still has: x = 10 (stale — the old value before the write)
```

### Case 1 — Choose Availability → Lose Consistency
- You decide *both requests should be processed* (write and read both work).
- WRITE `x = 20` arrives and is applied on Node A (Primary). x changes from 10 → 20. But because the link is broken, **replication to Node B cannot happen**.
- A READ comes → Node B responds → returns the **stale value x = 10** (the old value, before the write).
- The **latest write is not reflected on the read** → **Consistency is lost** (system stays available).

```
        Choose Availability (AP):
        ┌──────────────┐  ✗ BROKEN ✗  ┌──────────────┐
        │   Node A     │               │   Node B     │
        │  x = 20 ✓    │               │  x = 10 ⚠   │  ← stale!
        │  (updated)   │               │  (not synced)│
        └──────────────┘               └──────────────┘
        READ request comes → Node B responds → returns x = 10 (WRONG! latest is 20)
        System is AVAILABLE ✓  but NOT CONSISTENT ✗
```

### Case 2 — Choose Consistency → Lose Availability
- You decide to **fail the read request** during the partition rather than serve stale data.
- WRITE `x = 20` is applied on Node A. Because replication is broken, Node B still has the old `x = 10`.
- If a READ hits Node B now it would return `x = 10` — an **inconsistent (stale) read**.
- So you **turn off / reject reads from Node B** → return **"Failed" / Bad Request** to the client.
- Failing the request means **Availability is lost** (system stays consistent — no wrong data is ever served).

```
        Choose Consistency (CP):
        ┌──────────────┐  ✗ BROKEN ✗  ┌──────────────┐
        │   Node A     │               │   Node B     │
        │  x = 20 ✓    │               │  SHUT DOWN ⛔│  ← turned off
        │  (updated)   │               │  (to prevent │
        └──────────────┘               │  stale read) │
                                       └──────────────┘
        READ request comes → Node B says "Bad Request / Unavailable"
        System is CONSISTENT ✓  but NOT AVAILABLE ✗
```

**Conclusion:** Whenever a partition happens, **either Consistency is lost OR Availability is lost** — one of the two. This is what CAP Theorem says.

> Single node → no partition problem ever → CAP trade-off doesn't apply.

---

## 5. Types of Databases (Venn Diagram — pick 2 of 3)

**CAP & NoSQL databases:** NoSQL databases are great for distributed networks — they allow for **horizontal scaling** and can quickly scale across multiple nodes. So when deciding **which NoSQL database to use**, it is important to keep the **CAP theorem in mind**.

The three properties intersect pairwise: you can pick **CA**, **CP**, or **AP**.

```
        CAP — The Three Pairwise Choices

                          C (Consistency)
                              /\
                             /  \
                        CA  /    \  CP
                           /      \
                          /________\
               A (Availability)    P (Partition Tolerance)
                          \   AP   /

        CA = Consistency + Availability
             (sacrifice P → single node only; not for distributed systems)

        CP = Consistency + Partition Tolerance
             (sacrifice A → turn off stale nodes during partition)

        AP = Availability + Partition Tolerance
             (sacrifice C → serve stale data, sync later)

        All three (CAP) together: impossible in a distributed system.
```

### 5.1 CA Databases (Consistency + Availability)
- Provide both Consistency and Availability — but (as shown) C and A can't both hold **if a partition occurs**.
- So CA databases **don't get partitioned** → they are **not distributed** → on a **single node**.
- *"Unfortunately, CA cannot deliver fault tolerance in any distributed system."* In a distributed system, **partitions are bound to happen** (network links can always fail). Therefore **no truly distributed database can be CA** in practice.
- **Not a very practical choice** for distributed systems.
- **Examples:** relational databases like **MySQL, PostgreSQL** are often called "CA" — but this is a simplified classification. What it really means is that they are **designed for single-node or small-cluster use** where the designer assumes partitions won't happen. Once they are deployed as a distributed system (with replication across data centers), they face the same C-vs-A trade-off as any other distributed DB. The "CA" label reflects their default design goal, not an absolute guarantee in distributed deployments.
- Possible at small scale; **not practical at large scale**.

```
        CA (MySQL, PostgreSQL)
        ┌─────────────────────────────────────────────────────┐
        │  Single Node or Small Cluster                       │
        │  ┌──────────────────┐                               │
        │  │   MySQL DB       │  ← Consistent + Available     │
        │  │  (no partition   │    Works great at small scale │
        │  │   possible on    │    ⚠ Once distributed widely, │
        │  │   single node)   │      partitions WILL happen   │
        │  └──────────────────┘      → forced to choose C/A   │
        └─────────────────────────────────────────────────────┘
```

### 5.2 CP Databases (Consistency + Partition Tolerance)
- Provide **Consistency** and **Partition Tolerance**, but **NOT Availability**.
- **When a partition occurs:** *the system has to turn off the inconsistent node(s)* until the partition can be fixed.
  - Turning off Node B → **Availability is lost** (reads from it return Bad Request).
  - When the system recovers (partition fixed, communication re-established), normal work resumes.
- **Architecture:** *Only the Primary node receives all the WRITE operations in the given replica set. Secondary nodes replicate data from the primary, so if the primary fails, a secondary stands in.*
- **Example:** **MongoDB** — a NoSQL DBMS that uses **documents for data storage**. It is considered **schema-less** (does not require a defined database schema) and is **commonly used in big data and in applications running in different locations**.
- **Use case:** **Banking systems** — Availability is **not** as important; **Consistency is critical**.
  - Banks go down at night / for maintenance → availability matters less.
  - Data must be consistent (an account must not show ₹1 lakh on one node and ₹2 lakh on another).

```
        CP (MongoDB — Banking Use Case)
        ┌──────────────────────────────────────────────────────────┐
        │  Partition Occurs:                                       │
        │                                                          │
        │  ┌──────────────┐  ✗ BROKEN ✗  ┌──────────────┐        │
        │  │   Primary    │               │  Secondary   │        │
        │  │  (Running)   │               │  SHUT DOWN ⛔│        │
        │  │  balance=9k  │               │  (to prevent │        │
        │  └──────────────┘               │  stale read) │        │
        │                                 └──────────────┘        │
        │                                                          │
        │  User queries balance → Secondary says "Unavailable"    │
        │  CONSISTENT ✓  (no wrong data shown)                    │
        │  NOT AVAILABLE ✗ (during partition)                     │
        │                                                          │
        │  Why banking? You CANNOT show wrong balance!            │
        │  ₹10,000 on Node A vs ₹1,00,000 on Node B = disaster   │
        └──────────────────────────────────────────────────────────┘
```

### 5.3 AP Databases (Availability + Partition Tolerance)
- Provide **Availability** and **Partition Tolerance**, but **NOT Consistency**.
- **When a partition occurs:** do nothing special — let **both nodes keep working**. If the business logic allows temporary inconsistency, both READ and WRITE execute.
  - System is **inconsistent for a short while**; that's acceptable.
- When recovery happens (link re-established), **data replication occurs** and both nodes become consistent → this is called **Eventually Consistent**.
  - *"If a user tries to access data from a bad node, they won't receive the most updated version of data"* (e.g. a write changed `x` from 10 to 20 on Node A, but Node B was partitioned and still has `x = 10`; a read from Node B returns the stale `x = 10`) — but in AP databases this **doesn't matter** for a short period (not their target).
  - When the partition is resolved, *"AP data will sync the node to ensure consistency."*
- **Examples:** **Cassandra, DynamoDB**. (Apache **Cassandra** is a NoSQL database with **no primary node**, meaning all of the nodes remain available; it allows for eventual consistency because users can **re-sync their data right after a partition is resolved**.)
- **Use case:** **Facebook / social media** — we **value Availability more than Consistency**.

```
        AP (Cassandra, DynamoDB — Social Media Use Case)
        ┌──────────────────────────────────────────────────────────┐
        │  Partition Occurs:                                       │
        │                                                          │
        │  ┌──────────────┐  ✗ BROKEN ✗  ┌──────────────┐        │
        │  │   Node A     │               │   Node B     │        │
        │  │ likes = 500  │               │ likes = 499  │← stale │
        │  │  (updated)   │               │  (still up!) │        │
        │  └──────────────┘               └──────────────┘        │
        │           Both nodes keep running ✓                     │
        │                                                          │
        │  User sees 499 likes instead of 500 → Who cares?        │
        │  A 1-like difference on a FB post is acceptable!        │
        │                                                          │
        │  AVAILABLE ✓  NOT CONSISTENT (for now) ✗               │
        │                                                          │
        │  Partition heals → Nodes sync → 500 everywhere ✓        │
        │  This is called EVENTUAL CONSISTENCY                     │
        └──────────────────────────────────────────────────────────┘

        Eventually Consistent Timeline:
        t=0:  Partition occurs. Node A = 500 likes, Node B = 499.
        t=0:  Reads from B return 499 (slightly stale — acceptable)
        t=5s: Partition heals, replication happens
        t=5s: Both nodes = 500 likes ✓ (eventually consistent!)
```

---

## 6. Choosing a Database (Summary)

- **Banking system** → **CP** (consistency matters; uses MongoDB-style CP).
- **Social media / Facebook** → **AP** (availability matters; Cassandra, DynamoDB).
- **CA** → mostly only when you claim a partition will never happen → basically **single-node / impractical for distributed systems** (because partitions are inevitable in distributed systems).

```
        Decision Guide — Which DB to Choose?

        Ask: "What is more critical for my application?"

        ┌────────────────────────────────────────────────────────┐
        │                                                        │
        │  Is CONSISTENCY critical?                             │
        │  (Wrong data = disaster?)                             │
        │       YES → CP → MongoDB                              │
        │       Example: Banking systems                        │
        │                                                        │
        │  Is AVAILABILITY critical?                            │
        │  (App must stay up, stale data briefly is OK?)        │
        │       YES → AP → Cassandra, DynamoDB                  │
        │       Example: Social media (Facebook)                │
        │                                                        │
        │  Small scale, no distribution needed?                 │
        │       CA → MySQL, PostgreSQL                          │
        │       (Practically: single-node, partitions won't     │
        │        happen, so CA holds for now)                   │
        │                                                        │
        └────────────────────────────────────────────────────────┘
```

---

## 7. ACID vs BASE

- Banking systems follow **ACID** properties (must).
- Social networking systems should have **BASE** properties.
- **BASE** full form:
  - **BA** = **Basically Available**
  - **S** = **Soft State**
  - **E** = **Eventually Consistent**

```
        ACID vs BASE (Quick Overview):

        ACID (Strong guarantees — Banking)      BASE (Flexible — Social Media)
        ┌──────────────────────────┐            ┌──────────────────────────┐
        │ Atomicity   — all or     │            │ Basically Available      │
        │              nothing     │            │ — system always responds │
        │ Consistency — DB stays   │            │                          │
        │              valid       │            │ Soft State               │
        │ Isolation   — concurrent │            │ — state may change over  │
        │              safe        │            │   time without input     │
        │ Durability  — committed  │            │                          │
        │              data stays  │            │ Eventually Consistent    │
        └──────────────────────────┘            │ — consistency after some │
                                                │   time (not immediate)   │
                                                └──────────────────────────┘
```

---

## Quick Revision

- **C** = same data on all nodes; **A** = system always operational; **P** = tolerant of communication break between nodes.
- **CAP**: in a distributed system you can have only **2 of 3**.
- On a **partition**, it's a **trade-off between C and A**.
- **CA** = single-node / not practical for distributed (MySQL, PostgreSQL).
- **CP** = consistency + partition tolerance, sacrifices availability → **MongoDB**, **Banking**.
- **AP** = availability + partition tolerance, **eventually consistent** → **Cassandra, DynamoDB**, **Social media**.
- **ACID vs BASE** (BASE = Basically Available, Soft state, Eventually consistent).
