# 19 — Master Slave Architecture

> A general architecture in Computer Science.

---

## 1. Context

- Many lectures have focused on **scaling the database** because it is extremely important today.
- We **scale up only to a level**, then **scale out** (horizontal scaling) because **scale up is very costly / not a practical choice**.
- Reason: Internet is widespread, many mobile/client devices → number of requests is huge → must scale out the database.

---

## 2. The Need (Normal Architecture & Single Point of Failure)

- Normal architecture: multiple servers (S1, S2, S3) behind a **Load Balancer**.
- The **Load Balancer** interfaces all incoming requests and forwards each to whichever server is **least busy** (balances the load).
- Now suppose there is **only one DB** → **all requests go to this single DB**, which serves all servers as the database server.
- **This is a Single Point of Failure (SPOF).**
  - If this DB server crashes or is hacked → **the whole system is blocked**, availability becomes **zero**, all requests return as Bad Request / failure.

```
        Normal Architecture with Single DB — The SPOF Problem

        Users → Many Requests
               │
               ▼
        ┌──────────────────┐
        │   Load Balancer  │  ← distributes requests across servers
        └──────┬───────────┘
               │
        ┌──────┴──────────────────┐
        ▼          ▼              ▼
     ┌──────┐   ┌──────┐      ┌──────┐
     │  S1  │   │  S2  │      │  S3  │  ← App Servers (many, OK)
     └──┬───┘   └──┬───┘      └──┬───┘
        └──────────┴──────────────┘
                   │ ALL queries go here
                   ▼
          ┌─────────────────┐
          │  Single DB  ⚠   │  ← SINGLE POINT OF FAILURE!
          │  If this goes   │
          │  down → ENTIRE  │
          │  system fails!  │
          └─────────────────┘

        One DB crash = Zero availability for ALL users.
        This is unacceptable for production systems.
```

---

## 3. The Solution → Master Slave Architecture

- Add **another DB** — call it a **Copy / Replica**.
- Now requests get **divided** — some redirected to the original DB, some to the replica → load is **distributed**.
- If one goes down, some requests can still be served by the other.

**This architecture is the Master Slave Architecture:**
- One DB = **Master**
- Other DB(s) = **Slave**

```
        Master Slave Architecture — Solving SPOF

        Users → Many Requests
               │
               ▼
        ┌──────────────────┐
        │   Load Balancer  │
        └──────┬───────────┘
               │
        ┌──────┴──────────────────┐
        ▼          ▼              ▼
     ┌──────┐   ┌──────┐      ┌──────┐
     │  S1  │   │  S2  │      │  S3  │  ← App Servers
     └──┬───┘   └──┬───┘      └──┬───┘
        │          │              │
        └──────────┴──────────────┘
               │           │
        WRITEs │           │ READs
               ▼           ▼
        ┌────────────┐  ┌────────────┐  ┌────────────┐
        │   MASTER   │  │   SLAVE 1  │  │   SLAVE 2  │
        │  (Primary) │──►│ (Replica) │  │ (Replica)  │
        │  All writes│  │  Reads     │  │  Reads     │
        └────────────┘  └────────────┘  └────────────┘
              replication ──────────────────────►
        No SPOF! If Master fails → Slaves still serve READs.
```

### Ethical note (terminology)
- "Master/Slave" is used a lot in CS, but the term is considered **not ethically appropriate** (reflects the historical master–slave model), so some companies avoid it.
- Example: **GitHub** renamed the **master branch** to **main line / main**.
- Concept stays the same; the name "Master Slave" is still used in many places today.

---

## 4. How Master Slave Works (Roles)

- We **removed the Single Point of Failure**.
- **Master node / Master DB handles all WRITE operations.**
  - Why? Because the Master is the **Original DB** — also called the **Latest DB / Owner DB / Primary DB**.
  - The Master always has the **latest information**, so all writes go to it (only then can it remain latest).
- **Slaves handle ONLY READ operations.**
  - We **never** send WRITE operations to a slave.
  - Slaves **replicate** data from the Master (via DB replication) to get the latest updates.
- (This is the same as **Note 17, Pattern 3** — the cab-booking app scaling example: a Primary + Replicas, all WRITE requests go to Primary because it carries the latest updates, then we replicate to the slaves which only READ. That note didn't use the name "Master Slave".)

```
        Roles in Master Slave — Clear Separation

        ┌─────────────────────────────────────────────────────────┐
        │                                                         │
        │  WRITES (INSERT/UPDATE/DELETE)                         │
        │        │                                                │
        │        ▼                                                │
        │  ┌──────────────────────────────┐                      │
        │  │          MASTER              │                      │
        │  │  (Original / Latest / Owner) │                      │
        │  │  Has the most recent data    │                      │
        │  └──────────────────────────────┘                      │
        │        │         │         │   replication             │
        │        ▼         ▼         ▼                           │
        │  ┌──────────┐ ┌──────────┐ ┌──────────┐               │
        │  │  SLAVE 1 │ │  SLAVE 2 │ │  SLAVE 3 │               │
        │  │ READ ONLY│ │ READ ONLY│ │ READ ONLY│               │
        │  └──────────┘ └──────────┘ └──────────┘               │
        │        ▲         ▲         ▲                           │
        │        │         │         │                           │
        │  READs (SELECT) distributed across slaves             │
        │                                                         │
        └─────────────────────────────────────────────────────────┘
        Rule: WRITE → Master always. READ → Slaves always.
```

### Practical Diagram
- **Master DB** ← all WRITE requests (it must have the latest updates).
- **Slave DBs** → replicate from Master (time to time) → handle all READ operations.

---

## 5. How Does Replication Happen? (Sync vs Async)

There are **two ways** replication can happen:

### 5.1 Asynchronous Replication (Async)
- WRITE operations keep coming to the Master; slaves replicate **time to time** (after some time period).
- **Example timeline:** At `t = 0`, entry `x = 10` comes on Master. Slave1 updates `x = 10` at `t = 10`, Slave2 at `t = 12`. But the write happened at `t = 0`.
  - So between `t = 0` and `t = 10`, a READ on the slaves returns the **OLD value** of x (e.g. 20).
- Acceptable when the application **tolerates a small delay** in reads (like the cab-booking system). The delay is only a **few seconds** — not hours/days. After a short while, replicas must be updated.

```
        Asynchronous Replication Timeline:

        t=0:  WRITE x=10 → Master (updated immediately)
              Slaves: x=20 (OLD value, not yet synced)

        t=0 to t=10: Any READ → Slave returns x=20 (STALE)
                     This is called replication lag.

        t=10: Slave 1 syncs → Slave1: x=10 ✓
        t=12: Slave 2 syncs → Slave2: x=10 ✓

        t=12 onwards: All reads return x=10 (correct) ✓

        Master ──────────────────────────────────────────────►
          x=10 written at t=0

        Slave1 ─────────────────────────[syncs at t=10]──────►
          x=20 (stale)                    x=10 ✓

        Slave2 ──────────────────────────────[syncs t=12]────►
          x=20 (stale)                              x=10 ✓

        ✓ OK for: cab booking (driver location — few seconds lag is fine)
        ✗ NOT OK for: banking (stale balance is dangerous)
```

### 5.2 Synchronous Replication (Sync)
- Use when **delay is NOT acceptable** (e.g. **banking**).
- **Example (banking):** `balance = x - 1000` (was 10000 → should become 9000). If the client reads from a replica that has a 10-second delay, they'd still see the **old 10000**, which is **wrong**.
- **Sync flow:** When `x = x - 1000` request comes:
  - **Do not return success** until it is **confirmed replicated** to the slaves.
  - The Master applies the write (e.g. `balance = 9000`), then **propagates the change** to the replicas (either by forwarding the write operation or a log entry — the exact mechanism is called statement-based or row-based replication). Only after the replicas **acknowledge** that they have applied the change does the Master return success to the client.
- This guarantees **no stale reads** immediately after a write — required for banking-type applications. Trade-off: write latency is higher because the Master must wait for replica ACKs before responding.

```
        Synchronous Replication Flow:

        Client: WRITE balance = balance - 1000
                │
                ▼
        ┌──────────────────────────────────────────────────────┐
        │  Step 1: Write on Master (balance = 9000)            │
        │  Step 2: Immediately propagate to Slave 1 + Slave 2  │
        │  Step 3: Wait for Slave 1 ACK ✓                      │
        │  Step 4: Wait for Slave 2 ACK ✓                      │
        │  Step 5: ONLY NOW return SUCCESS to client           │
        └──────────────────────────────────────────────────────┘
                │
                ▼
        Client gets: "Transaction Successful" ✓
        At this moment: Master=9000, Slave1=9000, Slave2=9000
        → ALL consistent immediately, no lag!

        Trade-off: Slightly higher write latency (waiting for slave ACKs)
                   but ZERO risk of stale reads. Worth it for banking!

        Async vs Sync — When to use which?
        ┌──────────────────────┬───────────────────────────────┐
        │  Async               │  Sync                         │
        ├──────────────────────┼───────────────────────────────┤
        │  Small lag OK        │  Zero lag required            │
        │  Cab booking, feeds  │  Banking, healthcare, finance │
        │  Faster writes       │  Slightly slower writes       │
        │  Higher availability │  Stronger consistency         │
        └──────────────────────┴───────────────────────────────┘
```

> **It depends on the kind of system you are developing** (Async for delay-tolerant, Sync for strict/banking).

---

## 6. What if a WRITE (update) query arrives at a Slave?

**Option 1 — Never Allow (the original/correct model):**
- A Slave is **designed for READ ONLY**; it should **never accept WRITE queries**. (Ignore / never allow.)
- This is what keeps it a true **Master Slave model**.

**Option 2 — Allow it → it becomes Master-Master:**
- If you allow the slave to take a write, you must **propagate** that write back to the Master (write logic in a model / application layer / server).
- **But now this is NO longer a Master Slave model — it becomes a Master-Master model.**
- This relates to **Note 17, Pattern 4 — Multi-Primary configuration** (multiple masters arranged in a circular way, propagating writes among each other). You must figure out a way to propagate Master-to-Master.
- *If you allow a slave to take WRITE operations → it becomes Master-Master → it will no longer be Master Slave.*

```
        Option 1: Never Allow (True Master-Slave)

        ┌────────────┐     ┌────────────┐
        │   Master   │     │   Slave    │
        │  R + W ✓   │     │  READ ONLY │
        └────────────┘     │  WRITE ⛔  │  ← reject the write!
                           └────────────┘
        Clean separation maintained.

        Option 2: Allow WRITE on Slave → Becomes Master-Master

        ┌────────────┐  ←── sync ───►  ┌────────────┐
        │  Master A  │                  │  Master B  │
        │  R + W     │  ────── sync ──► │  R + W     │
        │            │  ◄──── sync ───  │            │
        └────────────┘   (circular)     └────────────┘
        Both can receive writes → Must propagate to each other
        → No longer Master-Slave → This is Multi-Primary (Pattern 4)

        ⚠ KEY RULE: If a slave accepts writes, it ceases to be a slave.
                    It becomes another master (Multi-Primary architecture).
```

> **Original architecture:** Master can only take WRITE operations; Slave takes READ operations.

---

## 7. Advantages of Master Slave Architecture

1. **Backup**
   - If the Primary/Master goes down, at least the **READ requests** can still be served by the Slaves.
   - WRITE requests are ignored for a while, but READs keep working → the system is **not fully failed** (only a portion fails). Slaves act as a **backup**.

2. **Multiple Masters possible**
   - If you need multiple masters → go to **Multi-Primary (Pattern 4)** in DB scaling.

3. **Scale Out READ Operations**
   - You scale out the READ operations. Good when the application has **few writes but many reads** (reads >> writes).
   - READ clients are scaled out → requests served with **minimum latency** / quick responses.

4. **Increased Availability & Reliability + Reduced Latency**
   - Makes the site reliable, increases availability, reduces latency.

5. **Parallelism**
   - READ requests **execute in parallel** — some reads sent to one slave, some to another → parallel execution.

```
        Advantages Illustrated:

        ┌──────────────────────────────────────────────────────────┐
        │                                                          │
        │  1. BACKUP:                                             │
        │     Master DOWN → Slaves still serve READs ✓           │
        │     System doesn't fully fail (partial service)        │
        │                                                          │
        │  2. SCALE OUT READs:                                    │
        │     100 READ requests → split across 3 slaves          │
        │     Each slave gets ~33 requests (fast!) ✓             │
        │     Perfect for read-heavy apps (social media, news)   │
        │                                                          │
        │  3. PARALLELISM:                                        │
        │     ┌──────────┐  ┌──────────┐  ┌──────────┐          │
        │     │  Slave1  │  │  Slave2  │  │  Slave3  │          │
        │     │ R1,R4,R7 │  │ R2,R5,R8 │  │ R3,R6,R9 │          │
        │     │ parallel │  │ parallel │  │ parallel │          │
        │     └──────────┘  └──────────┘  └──────────┘          │
        │     Reads execute simultaneously → low latency ✓       │
        │                                                          │
        │  4. AVAILABILITY ↑ + LATENCY ↓:                        │
        │     More nodes → more chances of fast response         │
        │     Any node failure → others compensate               │
        │                                                          │
        └──────────────────────────────────────────────────────────┘
```

---

## 8. Notes / Implementation Detail

- This is the **CQRS (Command Query Responsibility Segregation)** technique discussed in Note 17 (Pattern 3): when a single server can't handle efficiently, add **replicas**.
- Example used: **MongoDB** replicas (MongoDB replica set).
- **The Master DB and the Slave DB can be different data models** (e.g. Master could be SQL, slave could be MongoDB).
  - i.e. the Primary–Replica data model/architecture can differ — but then you must **write interfaces** for replication so it works as a proper application.
- Net effect: site **reliability, availability ↑**, **latency ↓**; you **scale out READ operations / traffic**.
- Depending on your application, you choose **synchronous updates** or not.
- Example referenced: Facebook scaling.

```
        Master-Slave with Different Data Models (Advanced):

        ┌──────────────────┐  replication  ┌──────────────────┐
        │  Master (MySQL)  │ ────────────► │  Slave (MongoDB) │
        │  SQL / Relational│  (via custom  │  NoSQL / Document│
        │  All WRITEs      │   interface)  │  All READs       │
        └──────────────────┘               └──────────────────┘

        WHY? MySQL is great for writes (ACID, transactions).
             MongoDB is great for fast reads (document store, flexible).
        Trade-off: You must write a custom replication interface.
        WHEN to use: When read and write patterns demand different DB models.

        Connection to CQRS (Note 17, Pattern 3):
        ┌─────────────────────────────────────────────────────┐
        │  CQRS = Command (Write) + Query (Read) Separated    │
        │  Master-Slave IS the physical implementation of CQRS│
        │  Master = Command side (writes)                     │
        │  Slave  = Query side (reads)                        │
        └─────────────────────────────────────────────────────┘
```

---

## Quick Revision

- **Single DB = Single Point of Failure (SPOF)** → solve by adding replicas → **Master Slave**.
- **Master** = original/latest/primary DB → handles **all WRITE** operations.
- **Slaves** = **READ only**; they **replicate** from the Master.
- **Replication:** **Async** (small delay OK, e.g. cab booking) vs **Sync** (no delay, return success only after replicas updated, e.g. banking).
- WRITE on a slave → **never allow** (keeps it Master-Slave); allowing it → becomes **Master-Master** (Multi-Primary, Pattern 4).
- **Advantages:** Backup, multiple masters, scale-out reads, ↑availability/reliability + ↓latency, parallelism.
- Tied to **CQRS / Note 17 Pattern 3**; Master & Slave can be different data models (needs interfaces).
