## 1. The Problem

- Suppose you have **one database** holding an application's entire information (a large service with **millions of customers**). There are **millions of requests daily**, worldwide, but a **single database.**
- A single DB system (one computer) — **no matter how much you vertically scale it** (make it a super-powerful supercomputer) — **cannot serve worldwide requests** efficiently; serving millions of requests (sequentially/synchronously) is very hard.
- **Second problem:** If that single database/computer **goes down** (hacker attack, maintenance, hardware failure), your **availability vanishes** — the website goes down, users see nothing.

→ To solve this, DBMS uses a concept that **arose mainly when the internet became very popular.**

```css
THE SINGLE DATABASE PROBLEM:

  Millions of users worldwide
  (India, US, UK, Japan...)
        │ │ │ │ │ │
        ▼ ▼ ▼ ▼ ▼ ▼
   ┌───────────────-──┐
   │  Single Database │  ← handles ALL requests sequentially
   │  (one computer)  │  ← if it goes down → website is DOWN
   └──────────────-───┘
           ↑
   Problems:
   1. Overloaded → slow responses (queues up millions of requests)
   2. Single point of failure → one crash = total outage

   No matter how powerful this computer is, it cannot serve
   the whole world efficiently from a single point.
```

---

## 2. Clustering(aks Replica Sets) — Concept & Logic

- **Clustering** (a.k.a. **Replica Sets / Replication**): create multiple servers — S1, S2, S3 — each holding a **replica** of the database.
- The databases are **identical**: `D1 = D2 = D3`. **Same data stored in all servers.** (Not "more users in S1, fewer in S2" — all servers hold the **same** dataset, e.g., the same 1000 users' info in each.)
- This **set of servers / replicas** is called a **cluster.** A cluster = a **bundle / group / collection** of replica servers.
- You **architect** your system so that you make the same instance, create **replica sets**, and store them as a **cluster.**

> **Key insight:** It's not about dividing data — it's about duplicating it across multiple servers so each server can independently handle any request.

**Where it's used:**
- Mostly in **NoSQL** databases (where internet, multiple servers, horizontal scaling, and website launches happen). Can be used in SQL too, but mostly NoSQL.
- **MongoDB** is a great example — it provides a **Replica Set** concept/function to create database replicas.

**Terminology note (server / node / host):**
- Think of a **server** as just a **capable computer** connected to the internet whose main job is to **store data**; requests come to it, it **processes and returns** the data.

```css
CLUSTERING / REPLICA SETS ARCHITECTURE:

  Before (single DB):           After (cluster of replicas):

  [Users] ──► [DB]              [Users]
                                    │
                                    ▼
                           ┌──────────────────┐
                           │  Load Balancer   │
                           └───┬──────┬───────┘
                               │      │      │
                         ┌─────▼┐ ┌───▼──┐ ┌▼─────┐
                         │  S1  │ │  S2  │ │  S3  │
                         │  D1  │ │  D2  │ │  D3  │
                         └──────┘ └──────┘ └──────┘
                         D1 = D2 = D3 (exact same data)

  All 3 servers hold identical copies of the database.
  This set of 3 servers = a CLUSTER.
```

---

## 3. Advantages of Clustering

### 3.1 Data Redundancy (the good kind)
- Hearing "redundancy" in DBMS may alarm you, but **not all redundancy is bad.** This is **NOT** the redundancy that caused anomalies (we are **not** repeatedly storing the same student "Rahul" within one table).
- Here, **the same user's info is replicated across different servers** — if Rahul's info is in S1, it's also in S2 and S3. This is a **different type of redundancy**: one server's data is **replicated** in the other servers.

```css
TWO KINDS OF REDUNDANCY — DON'T CONFUSE THEM:

  BAD redundancy (normalization problem):
  ┌──────────┬──────────┬──────┐
  │ Rahul    │ Delhi    │  CS  │ ← Rahul appears 3 times in
  │ Rahul    │ Delhi    │  CS  │   the SAME table → anomalies
  │ Rahul    │ Delhi    │  CS  │
  └──────────┴──────────┴──────┘

  GOOD redundancy (clustering):
  S1: { Rahul, Delhi, CS }    ← same data,
  S2: { Rahul, Delhi, CS }    ← different servers,
  S3: { Rahul, Delhi, CS }    ← backup for availability ✅
```

### 3.2 High Availability
- Suppose **S1 shuts down** (hacker attack detected → you shut it down; or regular maintenance; or a hardware problem).
- If there were **only one server / one replica**, the whole **website would go down.**
- With **2+ replicas holding the same info**, the **user never notices** — info that would have come from S1 simply comes from S2/S3.
- **Layer of abstraction:** Clustering adds a **layer of abstraction** — the user **doesn't know** whether data came from S1, S2, or S3; they just get their data. This is **hidden** from the user.
- → System becomes **robust and highly available** by introducing redundancy.
- **Formally:** If S1 is down, S2 & S3 are available as a **backup**, so at any point of time some server executes the request — and since all servers store the **same info**, the user always gets the **same data.**

```css
HIGH AVAILABILITY — S1 GOES DOWN:

  Normal operation:                  S1 crashes:
  ┌─────────────┐                    ┌─────────────┐
  │Load Balancer│                    │Load Balancer│
  └──┬───┬───┬──┘                    └────┬────┬───┘
     │   │   │                            │    │
    S1  S2  S3   ← all serving            S2  S3  ← S2,S3 take over
    ✅  ✅  ✅                             ✅  ✅   S1 ❌ (down)

  User experience: No difference! Requests still served.
  The abstraction layer hides which server is responding.
  Website stays UP ✅
```

### 3.3 Load Balancing
- Millions of requests may arrive. With a **single server**, all requests are handled **sequentially** → the server becomes **highly loaded** and may **hang or crash.**
- With **replica sets**, you **distribute requests**: some to S1, some to S2, some to S3 → **load is balanced** → no single system crashes; system runs robustly and the website/DB never goes down.

**Load Balancer (architecture component):**
- To do load balancing you must add a **logic / interface layer = the Load Balancer layer.**
- The load balancer **checks which server is least loaded** and **passes the new request to that server.**
- → If you have replica sets, you **need a load balancer** in your architecture to route each new request to the least-loaded server replica.

**Everything points toward Availability:**
- Load balancing → makes requests **fast** (a single loaded server adds delay) **and** prevents the single server from **crashing.** All of it ensures the **website stays ON at all times.**

```css
LOAD BALANCING — HOW IT WORKS:

  1000 simultaneous requests arrive:

  WITHOUT load balancing (single server):
  [req1,req2,...,req1000] ──► [S1: 100% loaded → queue → CRASH ❌]

  WITH load balancing (3 servers):
  ┌──────────────────────────────────────┐
  │           Load Balancer              │
  │  "S1 has 30 reqs, S2 has 28,         │
  │   S3 has 27 → send next to S3"       │
  └──────┬───────────┬────────────┬──────┘
         │           │            │
  ┌──────▼──┐  ┌─────▼───┐  ┌────▼────┐
  │ S1: 34% │  │ S2: 33% │  │ S3: 33% │  ← balanced!
  │ loaded  │  │ loaded  │  │ loaded  │
  └─────────┘  └─────────┘  └─────────┘

  No server is overloaded → all respond fast → no crash ✅
```

### 3.4 Scaling (enabled by Load Balancing)
- As user requests grow and a single server can't take the load, you **keep adding replica servers** and use the **load balancer** to handle the extra requests → **seamless scaling.**
- This also ensures **high availability** (one overloaded server might crash).

```css
SEAMLESS SCALING — add more replicas as demand grows:

  Year 1: 1M users         Year 3: 10M users
  [LB] → S1, S2, S3        [LB] → S1, S2, S3, S4, S5, S6, S7

  Just add more replica servers to the cluster.
  Load balancer automatically distributes across all of them.
  No downtime, no data migration — just add nodes ✅
```

---

## 4. How Clustering Works

- In a **clustering architecture**, all requests are **split across many computers/nodes**, so each individual user request is executed by some server.
- The **load balancer** looks at the incoming requests and routes each to whichever **server/node is least loaded.**
- → That's how the clustering architecture works.

```css
COMPLETE CLUSTERING ARCHITECTURE:

  ┌────────────────────────────────────────────────────────┐
  │                      USERS                             │
  │         (millions of requests globally)                │
  └────────────────────────┬───────────────────────────────┘
                           │
                           ▼
              ┌────────────────────────┐
              │     LOAD BALANCER      │  ← routes to least-loaded server
              │  (abstraction layer)   │  ← user never knows which server
              └────┬──────────┬────────┘
                   │          │          │
                   ▼          ▼          ▼
           ┌───────────┐ ┌──────────┐ ┌──────────┐
           │  Server 1 │ │ Server 2 │ │ Server 3 │
           │    DB     │ │   DB     │ │   DB     │
           │  (D1)     │ │  (D2)    │ │  (D3)    │
           └───────────┘ └──────────┘ └──────────┘
                D1   =    D2    =    D3  (identical data)

  Benefits: High Availability + Load Balancing + Scalability
```

---

## 5. CDN (Content Delivery Network)

A CDN is a geographically distributed caching layer that stores content on edge servers to reduce latency, decrease origin server load, and improve availability.

## Core Terminology

- **Edge Server:** A CDN server located physically close to users that serves cached content.
- **Origin Server:** The main backend server where the actual, authoritative content resides.
- **Cache Hit:** The requested content is found in the CDN cache and served immediately to the user.
- **Cache Miss:** The content is not in the cache. The CDN fetches it from the origin server, serves it, and stores a copy for future requests.
- **TTL (Time To Live):** A configuration setting that determines how long content remains in the cache before the CDN checks the origin for updates.
## Content Types
- **Static Content:** Images, CSS, JavaScript, HTML, and videos. Highly cacheable.
- **Dynamic Content:** User-specific data, live dashboards, or live API responses. Harder to cache, though CDNs accelerate this traffic via optimized routing and connection multiplexing.
## Primary Benefits

- Lower network latency.
- Faster page load speeds.
- Reduced bandwidth costs for the origin server.
- Enhanced scalability to absorb traffic spikes.
- Built-in protection against volumetric DDoS (Distributed Denial-of-Service) attacks.

## CDN vs. Load Balancer

|**Feature**|**CDN**|**Load Balancer**|
|---|---|---|
|**Primary Goal**|Reduces latency by moving content closer to users|Improves availability by distributing workload|
|**Core Function**|Caches and serves content from the edge|Routes incoming requests to backend servers|
|**Architecture**|Geographically distributed globally|Typically sits directly in front of an application cluster|

## System Design Concept: Geographic Distribution

The underlying principle of a CDN is identical to database clustering or replica sets: distribute the same data across multiple geographical regions so each user interacts with a local node.

The difference is the system layer. A CDN distributes temporary cached assets at the network edge. A database replica set distributes persistent, stateful data at the storage layer.

Plaintext

```css
      World Map (Geographic Distribution Principle)
      
       India PoP             US PoP               Europe PoP
       ┌──────────┐          ┌──────────┐        ┌──────────┐
       │ Edge     │          │ Edge     │        │ Edge     │
       │ Cache    │          │ Cache    │        │ Cache    │
       └──────────┘          └──────────┘        └──────────┘
            ▲                      ▲                    ▲
            │                      │                    │
       Indian user           US user              European user
       Nearest server →      Nearest server →     Nearest server →
       Low latency ✅        Low latency ✅        Low latency ✅
```

---

## Summary

| Term | Meaning |
|---|---|
| **Cluster** | A bundle/group/collection of replica servers working together |
| **Replica Set** | Group of servers each holding the **same** database (`D1 = D2 = D3`) so they work together |
| **Load Balancer** | Layer that routes each new request to the **least-loaded** server |
| **Abstraction Layer** | Hides from the user which server actually served the request |

### Advantages (recap)
1. **Data Redundancy** — same data replicated across servers (good redundancy, not anomaly-causing).
2. **Load Balancing** — distribute requests → no crash, faster responses, enables **seamless scaling**.
3. **High Availability** — if one server is down (maintenance/attack/failure), others serve the same data; website always ON.

### Key Takeaways
- **Clustering = making Replica Sets** = combining more than one server/instance, all connected to a **single (replicated) dataset.**
- Arose with the **popularity of the internet**; mostly used in **NoSQL** (e.g., **MongoDB Replica Sets**).
- All advantages converge on **High Availability** — a seamless experience so the website never goes down.
- **Real-world:** All big product-based companies use clustering in their internet-based products to ensure high availability.
- **CDNs** are geographically distributed replica sets for fast content delivery.
