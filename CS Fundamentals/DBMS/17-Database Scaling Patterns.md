## 0. Context & Setup

- This builds on earlier topics (Indexing, SQL/NoSQL, Sharding, Clustering, Partitioning, the Relational database) and moves to **Design** — how you **scale a database**.
- **When do we scale the database?** When the **user base grows** → **data grows** → **number of requests grows** → more users work **concurrently** on the same database.
- Studied **step by step**: when a startup is built, how small it starts, what scaling options apply, and at which point which scaling option fits as the user base grows.

**Goals of scaling (step by step):** increase **performance**, keep the database **available**, **no lag**, and a **good user experience**, since users and requests have grown. (Going forward we are building a **distributed database**.)

**Important real-world point — don't over-engineer at the start:**
- A new startup does **NOT** start with Amazon/Ola-style distributed design (multiple servers, multiple data centers in different continents).
- Reasons: (1) **no requirement yet**; (2) **cost** is a big matter — little revenue, money goes to **marketing/campaigns**; (3) your idea might **fail** / be unfavorable to the market right now.
- So you **start small** and scale **only as the requirement grows**.
- (Prerequisite: **Normalization** etc. well understood.)

```css
        The Golden Rule of Scaling:
        ┌─────────────────────────────────────────────────────────────┐
        │  DON'T over-engineer at the start.                         │
        │  Scale ONLY when you hit a real bottleneck.                │
        │                                                             │
        │  Startup Day 1        →  Single Machine (OK!)              │
        │  10 users/day         →  No problem                        │
        │  10,000 users/day     →  Optimize queries first            │
        │  100,000 users/day    →  Start scaling out                 │
        │  Millions/day         →  Sharding + Data Centers           │
        └─────────────────────────────────────────────────────────────┘
```

---

## The Running Example — Cab Booking App

- You build a small **Cab Booking app** (like Ola/Uber) in your locality.
- **Initial setup:** an **old/small machine** you had lying around, set up as a **single database server**. It stores: **customer information, trips information, locations information, booking data, trip history** — all together in one place.
- Few drivers, ~**10 customers** (friends / referral basis). Few trips (e.g. one trip in ~5 minutes). **Works fine.**
- App becomes famous via referrals → bookings rise to **~10 bookings per minute.**

```css
        Initial Setup (Works Fine)
        ┌───────────────────────────────────────────────────────┐
        │                                                       │
        │  Client App                                           │
        │       │                                               │
        │       ▼                                               │
        │  ┌──────────────┐      ┌──────────────────────────┐  │
        │  │  API Server  │─────►│   Single Old DB Server   │  │
        │  └──────────────┘      │  - Customer info         │  │
        │                        │  - Trips info            │  │
        │                        │  - Location info         │  │
        │                        │  - Booking data          │  │
        │                        │  - Trip history          │  │
        │                        └──────────────────────────┘  │
        │                                                       │
        │  Users: ~10   |  Bookings: 1 per 5 min  → Fine ✓     │
        └───────────────────────────────────────────────────────┘
```

---

## The Problem Appears

- App becomes **slow** — **latency**, late updates (writes update late, driver shows late), and sometimes the app **hangs**.
- Investigation: **API latency is the problem.**
- The small old machine could handle ~1 trip / 5 min, but now **10 bookings/min** → too many requests hit it → it **cannot process them in time** → for an X request, **response time shoots up** → app feels slow.
- **Real-life parallel:** changing your username (a **WRITE** goes through), but the change shows in **order history only an hour later** (the **READ** doesn't reflect it yet — explained later in CQRS).
- Symptoms: API latency ↑, transactions / **deadlocks / starvation**, frequent **failures**, many transactions **time out**, system **hangs**, late responses → **customers complain**.

```
        The Bottleneck Situation
        ┌──────────────────────────────────────────────────────────┐
        │                                                          │
        │  10 bookings/min arriving  ─────────────────────────►   │
        │                                                          │
        │  ┌────────┐  ┌────────┐  ┌────────┐   ...many more     │
        │  │ Req 1  │  │ Req 2  │  │ Req 3  │                    │
        │  └───┬────┘  └───┬────┘  └───┬────┘                    │
        │      └───────────┴───────────┘                          │
        │                  │ all hitting                          │
        │                  ▼                                       │
        │         ┌──────────────┐                                 │
        │         │  Small DB   │ ← overloaded!                   │
        │         │  CPU: 100%  │   deadlocks, timeouts           │
        │         │  slow disk  │   app hangs                     │
        │         └──────────────┘                                 │
        │                                                          │
        └──────────────────────────────────────────────────────────┘
```

**Solution path:** apply **performance optimization measures** first (so queries/requests process faster and user experience becomes reasonable for a human-acceptable latency), and ultimately **scale the system**. (Recall: an API layer + a DB getting N requests → add a replica → split into N/2 + N/2.)

---

## Pattern 1 — Query Optimization & Connection Pooling

> Applied **first** at low scale (~30 bookings/min). Don't directly increase the system's cost yet — go slow.

This pattern bundles several performance-optimization measures:

### 1.1 Caching
- **Cache frequently-used data** (store it in a fast memory; on a repeat request → **cache hit** vs **cache miss**).
- Cache **non-dynamic data** (data that won't change), e.g. **payment history / booking history of the last month, user profiles** — past bookings are fixed, so they're non-dynamic.
- **Dynamic data** must come from the DBMS/server, e.g. **driver's current location, current cost A→B, best route A→B, traffic status, which driver is where** — all dynamic, cannot be cached.

```css
        Without Cache                     With Cache
        ┌──────────────────────┐          ┌─────────────────────────────┐
        │ Request: Past trips? │          │ Request: Past trips?        │
        │   → Query DB         │          │   → Check Cache first       │
        │   → Disk read        │          │   → Cache HIT? Return fast! │
        │   → ~100ms           │          │   → Cache MISS? Query DB    │
        └──────────────────────┘          │   → Store in Cache          │
                                          │   → Next request = fast ✓   │
                                          └─────────────────────────────┘
        Cache WHAT: Payment history, booking history (non-dynamic)
        Don't cache: Driver location, current fare, live traffic (dynamic!)
```

### 1.2 Denormalization
- Normalization broke data into **many tables** (until redundancy was removed) → info is **spread across multiple tables**.
- A query needing such info must **hit all those tables and JOIN** them → **JOINs take a lot of time** → poor user experience. *(You learned in college that normalization is good and keeps the DB clean — but here it causes the problem.)*
- **Fix → Denormalization:** introduce **some controlled redundancy** — bring the two tables you had broken down back into one table (go a bit "up" from full normalization).
- Result: **fewer JOINs** → **query latency reduces / performance improves.**

```
        Normalized (Good for storage, Bad for speed):
        ┌──────────────┐    ┌─────────────┐    ┌──────────────┐
        │   Bookings   │    │  Customers  │    │   Drivers    │
        │  booking_id  │    │ customer_id │    │  driver_id   │
        │  customer_id │───►│ customer_nm │    │  driver_nm   │
        │  driver_id   │───────────────────────►  location    │
        └──────────────┘    └─────────────┘    └──────────────┘
        To show one booking: JOIN 3 tables → SLOW!

        Denormalized (Some redundancy, Much faster):
        ┌───────────────────────────────────────────────────────┐
        │                     Bookings                          │
        │  booking_id | customer_name | driver_name | location  │
        └───────────────────────────────────────────────────────┘
        To show one booking: 1 table query → FAST! ✓
        (Trade-off: some data is duplicated — acceptable here)
```

### 1.3 Use NoSQL
- Alternatively, **use a NoSQL database** instead of the relational DB.
- NoSQL's key strength: **related data is stored together in one package** → you get all needed data in one shot. Since the aim is **higher performance**, you either **introduce some redundancy** or **use NoSQL**.

### 1.4 Connection Pooling
- Every **DB connect** call creates a **new client→server communication/connection** over the internet → this is **costly in terms of time** (handshake, many protocols, uncertainty whether it establishes).
- **Fix:** instead of establishing a **new connection every time**, **maintain a pool of pre-created connections** (e.g. 4–5 connections / threads) and **reuse** them repeatedly.
- Benefit: the **latency/performance hit of repeatedly creating connections disappears.**
- Implemented via your framework (e.g. **Spring / Spring Boot**) connection-pool support — a very famous, commonly-used option.

```
        Without Connection Pooling            With Connection Pooling
        ┌──────────────────────────┐          ┌───────────────────────────┐
        │  Req 1: Connect → Query  │          │  Pre-created pool:        │
        │         (handshake ~50ms)│          │  [Conn1][Conn2][Conn3]    │
        │  Req 2: Connect → Query  │          │      (always ready)       │
        │         (handshake ~50ms)│          │                           │
        │  Req 3: Connect → Query  │          │  Req 1: Use Conn1 → Done  │
        │         (handshake ~50ms)│          │  Req 2: Use Conn2 → Done  │
        └──────────────────────────┘          │  Req 3: Use Conn3 → Done  │
          Every request pays the              │  No handshake cost! ✓     │
          connection setup cost               └───────────────────────────┘
```

**Outcome of Pattern 1:** caching + denormalization/NoSQL + connection pooling → **user experience improves**; customers are happy. Works fine **for the current limited load (~30 bookings/min).**

---

## Pattern 2 — Vertical Scaling (Scale Up)

> Triggered when you expand to **another city → ~100 bookings/min** → significant performance drop again (Pattern-1 optimizations were tuned for ~30/min and are now exhausted).

- You open Task Manager: **CPU ~80% busy**, **RAM ~89% full**, some things crash → the current system can't handle the requests.
- **Scale Up = increase the machine's capacity** — buy a good machine, increase **CPU / RAM / hardware**.
- You replace the **tiny old machine** with a much better one.

```css
        Pattern 2: Vertical Scaling (Scale Up)
        ┌────────────────┐                ┌────────────────────┐
        │  Old Machine   │                │    New Machine     │
        │  CPU:  2 core  │   Replace      │   CPU:  16 core    │
        │  RAM:  4 GB    │ ────────────►  │   RAM:  64 GB      │
        │  HDD:  500 GB  │   (buy better) │   SSD:  4 TB       │
        └────────────────┘                └────────────────────┘
         ~30 bookings/min                  ~100 bookings/min OK!

        ⚠ Limitation: Cannot scale up forever.
           More cores ≠ proportionally more performance.
           Cost increases rapidly (non-linear).
```

**Limitation:**
- Vertical scaling has a **limit** — you can scale up only to a point.
- It is **always costly** — the more you scale up, the more **cost increases.**

**Outcome:** With vertical scaling, the ~100 bookings/min sluggishness is fixed — API latency good, CPU fine, memory not full, DB runs well. Customers happy **for now.**

---

## Pattern 3 — CQRS (Command Query Responsibility Segregation) / Read Replicas

> Triggered when you expand to **3 more cities → ~300 bookings/min (3×)**. Problem returns: system needs **constant maintenance**, **indexing slows down** (re-indexing is slow), and the **cost of scale-up** is now too high (money already spent on growing the business). You can't apply Pattern 2 again.

**CQRS:**
- Idea: the request load isn't being handled → **separate READ requests from WRITE requests** onto **different physical machines**, instead of giving both read and write to one big machine.
- Take the initial machine and create **two replicas** of it (we have already studied **Clustering** — here we apply it neatly):
  - **Primary machine** → handles **all WRITE operations** (writes need transaction/**ACID** properties — a write must be successful).
  - **Replicas** → handle **all READ operations**.
- This separation is called **CQRS (Command Query Responsibility Segregation)** — the write **Command** and the read **Query** responsibilities are segregated.

```css
        CQRS — Separating READ and WRITE
        ┌──────────────────────────────────────────────────────────┐
        │                                                          │
        │  Incoming Requests                                       │
        │       │                                                  │
        │       ▼                                                  │
        │  ┌───────────┐                                          │
        │  │  Router   │ ← decides: is this a READ or WRITE?      │
        │  └─────┬─────┘                                          │
        │        │                                                  │
        │    ┌───┴───────────┐                                     │
        │    ▼               ▼                                     │
        │ WRITEs           READs                                   │
        │    │               │                                     │
        │    ▼               ▼                                     │
        │ ┌──────────┐   ┌──────┐ ┌──────┐ ┌──────┐              │
        │ │ PRIMARY  │──►│ Rep1 │ │ Rep2 │ │ Rep3 │              │
        │ │(Writes)  │   │(Read)│ │(Read)│ │(Read)│              │
        │ └──────────┘   └──────┘ └──────┘ └──────┘              │
        │   replication ────────────────────────────►              │
        │   (Primary pushes updates to ALL replicas)               │
        │                                                          │
        │  Command (Write) → Primary                               │
        │  Query  (Read)   → Replicas        [CQRS Pattern]       │
        └──────────────────────────────────────────────────────────┘
```

**Replication (Primary → Replicas):**
- The Primary **replicates data to the replicas time-to-time** (writes land on Primary, creating new changes).
- Example: `x = 10` on Primary. A write changes `x` to **20** on Primary → this `20` must **replicate** to the replicas (since reads go to replicas).

**The Replication-Lag question (very important):**
- Suppose Primary updated `x = 20`, but replication is time-to-time (say ~a minute later). Before replication completes, a **READ** hits a replica and reads the **old `x = 10`**.
- This is acceptable **if the business logic tolerates it**: *the write must be written properly; the read can become correct a moment later.*
- **Example:** food delivery / cab booking — the driver's car moves **slowly/gradually**; a small delay before the read reflects the latest position is fine.
- So the **read requests (N)** are divided across the read replicas (e.g. **N/3** across 3 read nodes), and writes go to the single Primary.

```css
        Replication Lag — Acceptable in Cab Booking

        t=0:  Driver moves → WRITE x=NewLocation → Primary updated
        t=0:  Read request → Replica still has OLD location (lag!)
        t=3s: Replication happens → Replica updated
        t=3s: Read request → NOW shows correct location ✓

        Is 3-second lag OK for cab booking? YES — car moves slowly.
        Is 3-second lag OK for banking?     NO  — money is critical.
```

**Outcome:** **all READ requests → replicas, all WRITE requests → Primary.** Performance improves; business runs well (~300 bookings/min).

---

## Pattern 4 — Multi-Primary Replication (Multi-Master)

> Triggered when you expand to **2 more cities → ~500 bookings/min**. Now **WRITE requests grew so much** that the **single Primary cannot handle all the writes** → the write (Primary) machine becomes slow → replication's acceptable lag also slows → user experience declines again.

- Idea: just as we made multiple replicas for reads, **make multiple Primaries** for writes → **Multi-Primary Replication** (with **multiple replicas also**).
- Create several nodes (e.g. **A, B, C, D**) where there is effectively **no fixed Primary/Secondary** — **every node can READ and WRITE**. All are **replicas of each other** in a cluster, each holding the **same full data**: same DB schema, same data, same indexes, same set of tables → **DB-A = DB-B = DB-C = DB-D** (copies of each other).

```css
        Multi-Primary (Multi-Master) Architecture
        ┌──────────────────────────────────────────────────────────┐
        │                                                          │
        │  Incoming Writes (distributed)                           │
        │  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐                  │
        │  │ W-1  │  │ W-2  │  │ W-3  │  │ W-4  │                  │
        │  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘                  │
        │     │         │         │          │                     │
        │     ▼         ▼         ▼          ▼                     │
        │  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐                  │
        │  │ DB-A │→ │ DB-B │→ │ DB-C │→ │ DB-D │→ (back to A)     │
        │  │ R+W  │  │ R+W  │  │ R+W  │  │ R+W  │                  │
        │  └──────┘  └──────┘  └──────┘  └──────┘                  │
        │      ↑_circular replication_______↑                      │
        │                                                          │
        │  Every node = same data = can READ and WRITE             │
        │  Writes distributed → no single overloaded master        │
        │  Reads: broadcast → whoever replies first wins           │
        └──────────────────────────────────────────────────────────┘
```

**How replication works — circular:**
- Replication happens in a **circle (Multi-Primary configuration)**: e.g. **B replicates from A, C replicates from B, D replicates from C**, and around — each node checks "did an update come?" and **propagates** it.
- Incoming **WRITE requests are distributed across the nodes** — any of the N requests can go to any node; pick whichever node is **least write-loaded** and give it the request. → writes are **distributed** → the previously overloaded single Primary's load is now spread → **better performance.**
- **READ:** reads are also **distributed across nodes** (routed to whichever node is least loaded, similar to how writes are distributed). Since all nodes hold the same full data and eventually sync via circular replication, any node can answer a read. (A small read/write lag is still expected and acceptable — but it must not exceed a level, e.g. the cab shouldn't still show the driver far away after he has actually reached your home.)

> **Outcome:** writes are distributed (`N` requests spread across nodes), reads are broadcast, and the current load is handled. Happy for now.

*(Note: conflict-resolution algorithms for circular Multi-Primary replication are out of scope here.)*

---

## Pattern 5 — Partitioning of Data by Functionality (Functionality-wise Partition)

> Triggered when you scale to **5 more cities → ~50 requests per second**. Business is super famous, but user experience worsens again even after all previous patterns. This is a **very famous** next optimization.

- You already had **Multi-Primary replication** (Pattern 4) in place. Now better to **partition the data by functionality**.
- Meaning: your database has **many tables**; certain **sets of tables together provide one functionality**, other sets of tables provide another functionality.
  - Example: some tables (with **longitude/latitude**) form a **location** database — these tables give **location information** at any point of time.
  - Similarly, another set of tables computes the **current cost of the whole journey**.
- So **separate** the location-related tables into their **own separate database**, on a **separate machine**. *(Note: the schema is changed — we logically split the DB schema functionality-wise.)*
- Put these separate DBs into **separate machines**, and each separate DB can itself have **Primary–Replica and Multi-Primary configuration** (since each can get heavy request load). e.g. the location DB on a separate machine, given Primary–Replica setup.
- **"Different DBs host data category by different functionality."** We divide the DB schema **logically, functionality-wise.**

```
        Before (One big DB — all tables together):
        ┌──────────────────────────────────────────────────────────┐
        │               One Monolithic DB                          │
        │  ┌────────────┐ ┌────────────┐ ┌───────────────────┐     │
        │  │  Location  │ │  Bookings  │ │  Customer/Trips   │     │
        │  │  Tables    │ │  Tables    │ │  Tables           │     │
        │  │ (lat/long) │ │            │ │                   │     │
        │  └────────────┘ └────────────┘ └───────────────────┘     │
        └──────────────────────────────────────────────────────────┘
        All requests compete for the same DB → slow

        After (Split by Functionality — each on its own machine):
        ┌───────────────────────────────────────────────────────────┐
        │  Location DB           Bookings DB       Customer DB      │
        │  ┌──────────────┐    ┌──────────────┐  ┌─────────────┐    │
        │  │ lat/long     │    │ booking_id   │  │ customer_id │    │
        │  │ driver_pos   │    │ trip_details │  │ trip_history│    │
        │  │ routes       │    │ fare info    │  │ payments    │    │
        │  │ (P+Replicas) │    │ (P+Replicas) │  │(P+Replicas) │    │
        │  └──────────────┘    └──────────────┘  └─────────────┘    │
        │      Machine 1           Machine 2          Machine 3     │
        └───────────────────────────────────────────────────────────┘
        Each DB is independently scalable! ✓
```

**The problem this introduces:**
- Suppose the location-specific data (e.g. **2 GB**) is moved to its own **Location DB**; the two DBs are now **independent of each other**.
- Earlier the client got the full answer from one DB. Now **some information is in one DB and some in another.**
- The **client application (or a back-end layer)** must now do a **logical JOIN / merge**: send some requests here, some there, **handle both, merge the data logically, then output** (e.g. build a JSON from the merged results and return it).
  - Example: a location-specific query needs **customer info from one DB** and **location info from another DB** → run queries in both places → **logically join the results** → output.
- This is a **complex problem** done on the *backend server* / *application layer*. **Cost:** responsibility of joining results shifts to your application/back-end layer (slightly more work).

```css
        How the App Layer Handles Cross-DB Queries:

        Client: "Show me trip summary with driver location"
               │
               ▼
        ┌────────────────────┐
        │   App/API Layer    │
        │  1. Query LocationDB → get driver lat/long
        │  2. Query BookingsDB → get trip details
        │  3. Merge results  → build JSON response
        │  4. Return to client
        └────────────────────┘
        The app layer acts as the JOIN operator — extra complexity, but worth it.
```

---

## Pattern 6 — Sharding (Horizontal Scaling / Scale Out by rows)

> Triggered when, after targeting **20–25 cities in your country**, you **expand to another country** (e.g. **Singapore / New York**). Your whole system (DBs up to Pattern 5) still sits **in India**. Requests from the new country travel to the India server → **high latency** (country-wise problem). So you **horizontally scale / scale out.**

**Sharding:**
- Apply **Sharding** = **horizontal partition** of the data.
- Example: take the database and **establish ~50 machines**. Configure them with the **same DB schema** (like the Student-table example in the Sharding notes), **but each machine holds just a part of the data.**
- Initially one machine had **N rows**; now across 50 machines you put, e.g., **50 rows per machine** — i.e. you **divide the rows across many machines.** *"Each machine holds just a part of the data."*

```
        Sharding — 50 Machines, Same Schema, Different Data

        Before (1 machine, N rows):
        ┌────────────────────────────────┐
        │   DB Machine (India)           │
        │   Row 1   … Row N              │
        │   (ALL data in one place)      │
        │   Singapore/NY users → high    │
        │   latency due to distance!     │
        └────────────────────────────────┘

        After (50 machines, N/50 rows each):
        ┌──────────┐ ┌──────────┐  ┌──────────┐      ┌──────────┐
        │ Shard 1  │ │ Shard 2  │  │ Shard 3  │ ...  │ Shard 50 │
        │ rows     │ │ rows     │  │ rows     │      │ rows     │
        │ 1–20k    │ │ 20k–40k  │  │ 40k–60k  │      │ 980k–1M  │
        │+replicas │ │+replicas │  │+replicas │      │+replicas │
        └──────────┘ └──────────┘  └──────────┘      └──────────┘
              ▲ same schema on all shards, different rows
              ▲ each shard has its own replicas for failure recovery
```

**Important requirements:**
- **Locality of data** must be maintained — local data should stay within one machine so you don't have to **switch between machines** repeatedly.
- **Each machine can have its own replicas** — used for **failure recovery**.
- Sharding has its own **problems** (discussed in the Sharding notes).

**Outcome:** after scaling out (e.g. 50 machines), the system runs well in the new regions (Singapore, New York). The **requests get divided across the 50 machines** (each holds some data), and each machine has its own replica system for failure recovery → latency reduces.

---

## Pattern 7 — Data-Center-wise Partition (across Continents)

> Triggered when the business becomes **global / mature** across **multiple countries and continents** (cities in Europe too). The Sharding (Pattern 6) machines were all set up **in India**; with worldwide requests, **throughput decreases, latency increases, lag returns** — Indian users are fine, but users in other countries get a poor experience.

- Because cross-continent requests travel with **very high latency**, **distribute the traffic across Data Centers.**
- Build **data centers in different regions**:
  - **Data Center 1 — Europe**
  - **Data Center — Singapore / India** (South Asia)
  - **Data Center — New York / USA**
- **Route requests continent-wise:**
  - Europe + Russia requests → **Europe data center**
  - South Asia requests → **Singapore / India data center**
  - USA requests → **American data center**
- You can add more data centers similarly. So data is **distributed across countries**, and requests are **divided across continents**.

```
        Pattern 7: Data-Center-wise Partition (Global View)

        Europe Users        Asia Users          USA Users
              │                  │                   │
              ▼                  ▼                   ▼
        ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
        │  DC Europe   │  │ DC Singapore │  │  DC New York │
        │  (Sharded +  │  │  / India     │  │  (Sharded +  │
        │   Replicas)  │  │  (Sharded +  │  │   Replicas)  │
        └──────┬───────┘  │   Replicas)  │  └──────┬───────┘
               │          └──────┬───────┘          │
               │                 │                   │
               └────────── cross-DC replication ─────┘
                    (all 3 DCs replicate each other)
                    If Europe DC fails → route Europe users
                    to Singapore/India DC temporarily

        User in UK      → Europe DC      → low latency ✓
        User in Mumbai  → Singapore DC   → low latency ✓
        User in Texas   → New York DC    → low latency ✓
```

**The availability problem & fix (cross-DC replication):**
- If we keep only Europe Data in Europe Server then if it goes down we cannot redirect requests.
- Suppose the **Europe data center fails** (down for an hour or two). Then your famous cab-booking app's **availability becomes zero** for that region — but we want **high availability** (studied earlier).
- **Fix:** keep **cross-data-center replication** — data centers **replicate among themselves time-to-time.**
- So if the **Europe DC goes down** for some reason, you can **redirect all of Europe's requests to (e.g.) India** → **availability stays high.** When the **Europe DC revives**, you **redirect those requests back to Europe.**

```css
        Cross-DC Failover (Availability Guarantee):

        Normal:    Europe Users → Europe DC ✓
        DC Fails:  Europe DC ⚠ DOWN
                   Europe Users → redirected to → India DC
                   (latency slightly higher, but app WORKS!)
        DC Revives: Europe DC ✓ UP again
                   Europe Users → redirected back to → Europe DC
```

**Outcome:** With 7 design patterns, you took the app from a **small locality to world level** — availability maintained worldwide, business thriving in every country → you become a **unicorn / worldwide brand.**

---

## Summary — The 7 Database Scaling Patterns (Cab-Booking Journey)

| # | Trigger (load) | Pattern | What was done |
|---|----------------|---------|---------------|
| — | very few users | **Single machine, single DB** | Stores customers, trips, locations, bookings, history together |
| 1 | ~30 bookings/min | **Query Optimization & Connection Pooling** | Caching (non-dynamic data), **Denormalization**, **NoSQL**, **Connection Pooling** (Spring/Spring Boot) |
| 2 | ~100 bookings/min | **Vertical Scaling (Scale Up)** | Increase machine CPU/RAM; limited & costly |
| 3 | ~300 bookings/min | **CQRS** | Separate READ/WRITE → **Primary for writes, Replicas for reads** (replication lag tolerated) |
| 4 | ~500 bookings/min | **Multi-Primary Replication** | Multiple read+write nodes, **circular** replication; writes distributed, reads broadcast |
| 5 | ~50 requests/sec | **Partition by Functionality** | Split DB schema functionality-wise (e.g. Location DB) onto separate machines; client/back-end does logical **JOIN/merge** |
| 6 | expand to other countries | **Sharding** | Horizontal partition — ~50 machines, **same schema, each holds part of data**; locality + per-machine replicas for failure recovery |
| 7 | global / cross-continent | **Data-Center-wise Partition** | Data centers per region; route requests continent-wise; **cross-DC replication** for high availability |

```
        The Full Journey at a Glance:

        10 users         → Single Machine
             ↓ grows
        30 bookings/min  → P1: Cache + Denormalize + Connection Pool
             ↓ grows
        100 bookings/min → P2: Vertical Scale Up (buy better hardware)
             ↓ grows
        300 bookings/min → P3: CQRS (Primary=Write, Replicas=Read)
             ↓ grows
        500 bookings/min → P4: Multi-Primary (all nodes Read+Write)
             ↓ grows
        50 req/sec       → P5: Partition by Functionality (Location DB, etc.)
             ↓ grows
        Expand globally  → P6: Sharding (50 machines, same schema, split rows)
             ↓ grows
        Worldwide        → P7: Data Centers per continent + Cross-DC Replication
```

**Recap of where Clustering shows up:** Clustering (a basic concept) is applied throughout — in Pattern 3 (Primary–Replica, read/write divided), in Pattern 4 (Primary–Replica/Multi-Primary, but **not** divided on read/write basis), and the later patterns build on it.

---

## Takeaway

- The scaling order, **step by step**: first query optimization & connection pooling, then vertical scaling, then CQRS (read replicas), then multi-primary for write-heavy load, then partition by functionality, then sharding (horizontal), and finally data-center-wise partition with cross-DC replication for global availability.
- This applies **all concepts together** in the context of building a startup.

---

## Quick Revision

- Scale **only when required** (cost & requirement; don't over-engineer at the start).
- **P1**: Caching (non-dynamic) + **Denormalization** / **NoSQL** (fewer JOINs) + **Connection Pooling** (reuse connections).
- **P2 Vertical Scaling**: ↑CPU/RAM — limited & costly.
- **P3 CQRS**: Primary = writes, Replicas = reads; **replication lag** tolerated if business allows.
- **P4 Multi-Primary**: many read+write nodes, **circular** replication; writes distributed, reads broadcast.
- **P5 Partition by Functionality**: split schema by functionality onto separate machines; client/back-end **merges (logical join)**.
- **P6 Sharding**: horizontal — many machines, same schema, **each holds part of data**; locality + per-machine replicas.
- **P7 Data-Center-wise Partition**: regional data centers, continent-wise routing, **cross-DC replication** → high availability worldwide.
