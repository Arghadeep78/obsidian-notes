# Low Level Design (LLD) — Introduction Notes

### SDE-1 Interview Prep

---

## 1. What is LLD?

**Low Level Design (LLD)** is the process of designing the **code structure** of an application — defining its objects, their relationships, interactions, and how they work together to form a complete, working system.

> **Key insight:** LLD is NOT about writing algorithms in isolation. It's about designing the _skeleton_ of an application around which algorithms (DSA) plug in.

### The Simple Mental Model

|Layer|Role|Analogy|
|---|---|---|
|**HLD** (High Level Design)|System architecture — servers, databases, infra|Blueprint of a building|
|**LLD** (Low Level Design)|Code structure — classes, objects, relationships|Room layout & interior structure|
|**DSA**|Algorithms that power individual operations|The electrical wiring|

> **Golden Rule:** _If DSA is the brain of an application, LLD is its skeleton._

---

## 2. DSA vs LLD — What's the Difference?

A classic mistake freshers make: jumping straight to algorithms without designing the application first.

### The Anurag vs Maurya Story

Imagine two developers given the task: _"Build QuickRide — an Ola/Uber-style platform."_

**Anurag (knows DSA, no LLD):**

- Immediately jumps to algorithms
- Problem 1 → Shortest path (Dijkstra on city graph)
- Problem 2 → Nearest driver assignment (Min-Heap / Priority Queue)
- Goes to manager with these solutions

**Manager's response:** _"These are just algorithms, not an application."_

- What are the objects/entities in the system?
- How do they interact with each other?
- How do we handle data security (e.g., hiding phone numbers after ride ends)?
- How do we integrate notifications and payment gateways?
- How does the app scale to millions of users?

**Maurya (knows DSA + LLD):**

- First identifies **entities**: `User`, `Rider`, `Location`, `Notification`, `Payment`
- Designs **relationships** between them (how User interacts with Rider, Rider with Location, etc.)
- Considers **data security** (e.g., phone number visibility rules)
- Plans for **scalability** (millions of concurrent users)
- _Then_ applies DSA (Dijkstra, Min-Heap) to solve specific sub-problems

**Takeaway:** LLD comes before DSA in the application-building process. DSA is a tool that LLD uses.

---

## 3. What LLD Focuses On

LLD has three core pillars:

### 3.1 Scalability

> Can your application handle millions of users without breaking?

- How does the system sustain high load?
- How easily can new features be added without a complete rewrite?
- The code/architecture should expand gracefully under load.

### 3.2 Maintainability

> Is your code easy to update, extend, and debug?

- Adding a **new feature** should NOT break existing features
- Bugs should be easy to isolate and fix (debuggable code)
- Aim for **minimum bugs** — no application is 0% bug-free, but good LLD minimizes them
- Rule of thumb: _If adding Feature X causes Feature A, B, C to break — your LLD is bad._

### 3.3 Reusability

> Can modules be picked up and used in other applications without major changes?

- Code should NOT be tightly coupled to one specific application
- Example: A **Notification** module or **Payment Gateway** module should be plug-and-play across QuickRide, Swiggy, Zomato, etc.
- A **Rider Matching Algorithm** is not QuickRide-specific — the same logic applies to food delivery (Zomato), package delivery (Amazon, Blinkit), etc.
- **Avoid tightly coupled code** — your modules should be independently portable

> **Plug-and-Play Model:** Write code such that any module can be lifted out of one application and dropped into another with minimal changes.

---

## 4. What is NOT LLD (i.e., HLD)

**High Level Design (HLD)** / System Design deals with:

|Concern|Example|
|---|---|
|**Tech stack**|Java + Spring Boot, Node.js, etc.|
|**Database choice**|SQL vs NoSQL vs Hybrid|
|**Server scaling**|Horizontal/vertical scaling, auto-scaling groups|
|**Cost optimization**|Scale servers up when traffic spikes, down when it drops|
|**Infrastructure**|Load balancers, CDNs, message queues, caches|

In an HLD interview, you write **almost zero code** — you're drawing architecture diagrams: servers, databases, queues, and how data flows between them.

In an LLD interview, you write **actual code** — class diagrams, object relationships, design patterns.

---

## 5. The Full Picture — HLD + LLD + DSA Together

```
User Request
     │
     ▼
┌─────────────────────────────────┐
│         HLD / System Design      │  ← Which servers? Which DB? How to scale?
│  (Infrastructure & Architecture) │
└────────────────┬────────────────┘
                 │
                 ▼
┌─────────────────────────────────┐
│     LLD / Low Level Design       │  ← Which classes? How do objects interact?
│  (Code Structure & Architecture) │     What design patterns to use?
└────────────────┬────────────────┘
                 │
                 ▼
┌─────────────────────────────────┐
│     DSA / Algorithms             │  ← Dijkstra for shortest path,
│  (Problem-Solving Logic)         │     Min-Heap for nearest driver, etc.
└─────────────────────────────────┘
```

All three work **together** to build a complete application. None is sufficient alone.

---

## 6. Why LLD Matters for FAANG/Top Startup Interviews

- **FAANG SDE-1 rounds** include at least one LLD round (design a parking lot, design a snake-and-ladder game, etc.)
- Open source contributions require understanding a codebase's LLD
- Working in teams demands maintainable, extensible code — that's LLD thinking
- HLD (system design) itself requires strong LLD foundations — you can't design a scalable system if you don't know how to write scalable code

---

## 7. Prerequisites for LLD

- **OOP (Object-Oriented Programming)** — mandatory. LLD is built on OOP principles.
- **SOLID Principles** — the backbone of good LLD
- **Design Patterns** — Creational, Structural, Behavioral (covered later in this series)
- Basic DSA knowledge (you'll need it to implement specific operations)

---

## 8. Series Roadmap

```
LLD Introduction  (this lecture)
       │
       ▼
OOP Pillars (Encapsulation, Abstraction, Inheritance, Polymorphism)
       │
       ▼
SOLID Principles
       │
       ▼
Design Patterns (Singleton, Factory, Observer, Strategy, etc.)
       │
       ▼
Case Studies (Parking Lot, BookMyShow, Splitwise, Snake-Ladder, etc.)
```

---

## 9. Quick Revision Cheatsheet

|Term|One-line definition|
|---|---|
|**LLD**|Designing code structure: classes, objects, relationships|
|**HLD**|Designing system architecture: servers, DBs, infra|
|**DSA**|Algorithms for solving isolated sub-problems|
|**Scalability**|System handles growing load gracefully|
|**Maintainability**|New features don't break old ones; easy to debug|
|**Reusability**|Modules work across multiple applications (plug-and-play)|
|**Tightly Coupled**|Code so tied to one application it can't be reused — BAD|
|**Loosely Coupled**|Modules are independent and interchangeable — GOOD|

---

## 10. Interview Tips

1. **Don't jump to algorithms first** — identify entities and their relationships first (be Maurya, not Anurag)
2. **Always think about extensibility** — "What if we need to add feature X tomorrow?"
3. **Think in terms of objects** — who are the actors? what data do they hold? what can they do?
4. **Mention design patterns** — even if not asked, show you know _why_ you chose a certain structure
5. **Scalability is not just HLD** — write code that is easy to scale (e.g., avoid hardcoding, use interfaces)

---

_Source: LLD Series — Lecture 1 (Introduction to LLD)_