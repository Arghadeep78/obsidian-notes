# What is Deadlock

## Background: Why This Topic Comes Up

- The last three lectures covered classic synchronization problems.
- When threads synchronize using locks on critical sections — thread T1 locks a critical section, T2 cannot enter and has to wait — this mutual exclusion is the root of a class of bugs called **deadlock**.

---

## System Resources

- A computer system has a **finite number of resources**: memory space, CPU cycles (if two CPUs exist, there are two instances of that resource), files, locks, I/O devices, etc.
- Multiple **processes** compete to use these resources in order to execute.
- Inside each process there can be multiple **threads** (T1, T2, …).
- For the purpose of discussing deadlock, the terms *process* and *thread* are used **interchangeably** — everything stated for processes applies equally to threads.

**The system's goal:** Manage finite resources across multiple processes such that the system keeps making forward progress. Each process waits a bit, gets the resource, uses it, releases it — the system keeps running and never hangs (no blockage).

---

## What Is Deadlock? (Motivating Example)

**Setup:**
- Two resources: R1, R2 (can be anything — memory, a lock, an I/O device).
- Two processes: P1, P2.
- P1 needs **both R1 and R2** to complete its execution.
- P2 needs **both R1 and R2** to complete its execution.
- P1's code is written such that it cannot complete unless it gets both resources together.

**What happens:**
- R1 gets allocated to P1 — P1 locks R1.
- R2 gets allocated to P2 — P2 locks R2.
- P1 is now waiting for R2 (which P2 holds).
- P2 is now waiting for R1 (which P1 holds).
- *Neither can move forward.* Both are stuck waiting, and this wait will continue **forever** — a circular dependency has formed.

> *The system is now hung. Neither P1 nor P2 can advance its program counter. This infinite wait is called a **deadlock**.*

Deadlock is dangerous because:
- From the user's perspective, the PC has hung — nothing is happening.
- CPU cycles (a precious resource) are completely wasted.
- No progress is being made by any involved process.
- The processes involved never finish executing, and the system resources are tied up — preventing other jobs from starting.

---

## Formal Definitions of Deadlock

**Definition 1:**
> A process requests a resource R. If R is not available (taken by some other process), the process enters a waiting state. Sometimes that waiting process is **never** able to change its state because the resource it has requested is **busy forever**. That is called **deadlock**.

**Definition 2:**
> Two or more processes are waiting for some resource availability which will **never** become available, because it is also held by some other process that is itself waiting. This is called **deadlock**.

- Deadlock is a **bug** in process and thread synchronization.
- *Both processes and threads can cause deadlock — the word "process" is used throughout, but it applies identically to threads.*

---

## How a Process Utilizes a Resource (3 Phases)

Every time a process/thread uses a resource, it goes through exactly three phases:

```css
1. Request  →  2. Use  →  3. Release
```

1. **Request:** The process checks whether the resource is available by checking if its lock is free. If the lock is free, the process locks the resource (acquires it). In the RAG diagram, this is shown as the resource being *allocated* to the process (an assigning edge from the resource to the process). No other process can use it now.
2. **Use:** The process uses the resource exclusively. Because it holds the lock, no other process can access the resource during this time.
3. **Release:** When the process is done, it releases (unlocks) the resource. Once released, any other process can acquire it.

---

## Necessary Conditions for Deadlock (Coffman Conditions)

There are exactly **four** necessary conditions. *All four must hold simultaneously* for deadlock to occur. If even one of the four is absent, deadlock cannot happen.

> These four conditions are directly asked in interviews: *"What are the necessary conditions for deadlock to occur in a system?"*

### Condition 1: Mutual Exclusion

- At any given time, only **one** process or thread can use a particular resource.
- If resource R is allocated to P1 and P2 requests R, P2 must **wait** until P1 releases R.
- This is enforced by locking — the mechanism we use to protect critical sections.

### Condition 2: Hold and Wait

- A process is holding at least one resource (has locked it) **and** is simultaneously waiting for another resource that is currently held by a different process.
- Example: P1 holds R1 (has locked R1) and is waiting for R2 (which P2 holds). P1 is both holding and waiting.

### Condition 3: No Preemption

- A resource cannot be forcibly taken away from the process holding it.
- The resource is released **only voluntarily** by the process, after it has finished using it — when its own code says "release."
- The OS cannot go to a process and forcibly reclaim a resource that was already allocated to it.
- Formally: *"Resource must be voluntarily released by the process after completion of execution."*

### Condition 4: Circular Wait

- A **circular chain** of processes exists: P1 is waiting for a resource held by P2, P2 is waiting for a resource held by P3, …, Pn is waiting for a resource held by P1.
- This is an extended version of Hold and Wait — it is Hold and Wait in a circular fashion.
- Example: P1 holds R1 and waits for R2; P2 holds R2 and waits for R1 → circular dependency.

### Verifying All Four Conditions — Worked Example

**Scenario:** T1 holds R1 and requests R2. T2 holds R2 and requests R1.

| Condition | Status | Reason |
|---|---|---|
| Mutual Exclusion | ✓ True | The resource allocation code is written so that when any thread locks a resource, no other thread can use it |
| Hold and Wait | ✓ True | T1 is *holding* R1 and *waiting* for R2. T2 is *holding* R2 and *waiting* for R1 |
| No Preemption | ✓ True | R1 will not be released until T1's own code says to release it; T2 cannot forcibly take R1 from T1 |
| Circular Wait | ✓ True | T1 → waiting for R2 (held by T2) → T2 → waiting for R1 (held by T1) → back to T1 |

*Since all four hold simultaneously → the system is in deadlock.*

---

## Resource Allocation Graph (RAG)

The **Resource Allocation Graph** (RAG, also written as RAG) is a standard pictorial representation of the system's current state — which resources are allocated to which processes, and which processes are requesting which resources. We have already been drawing RAGs throughout the previous examples; this section formalises the notation.

### Vertices (Nodes)

```css
Process vertex:   drawn as a circle    →   ( P1 )
Resource vertex:  drawn as a box       →   [ R1 ]
```

### Instances of a Resource

A resource can have **multiple instances**. For example:
- A quad-core CPU is one resource ("CPU") with **four instances** (four cores).
- Two monitors connected to the same machine → two instances of "monitor."
- Two hard disks → two instances of "disk."

How instances are shown inside the resource box:
```css
[ • ]         →  1 instance (single instance — implied if no dots drawn)
[ • • ]       →  2 instances
[ • • • • ]   →  4 instances (e.g., quad-core CPU)
```

If a resource box just has a label and no dots, it means **single instance**.

### Edges

```css
Assigning edge (allocation edge):
    R ──→ P    (resource R has been allocated to process P)
    Drawn FROM the resource TO the process.

Request edge:
    P ──→ R    (process P is requesting resource R)
    Drawn FROM the process TO the resource.
```

### Why RAG Is Useful

The RAG lets us take a bird's-eye view of the entire system state and visually detect **cycles**. The cycle detection connects directly to the fourth necessary condition (circular wait).

### RAG Rule for Deadlock Detection

> - **No cycle in RAG → No deadlock (guaranteed).**
> - **Cycle in RAG → Deadlock *may* exist (not guaranteed).**

The reason for "may" is explained by the two examples below.

---

### RAG Example 1: Cycle Present → Deadlock IS Present

**System state:**
- R1 allocated to P2. P1 is requesting R1.
- P2 is requesting R3. R3 is allocated to P3.
- P3 is requesting R2. R2 has multiple instances, but the one instance of R2 relevant here is already allocated to P2.
- Two cycles are visible in this graph.

```css
(P1) ──requests──→ [R1] ──allocated──→ (P2)
(P2) ──requests──→ [R3] ──allocated──→ (P3)
(P3) ──requests──→ [R2] ──one instance allocated──→ (P2)
```

- Both cycles involve P2 and P3 waiting on each other's held resources.
- Verifying the four conditions:
  - *Mutual exclusion:* ✓ (locks are used).
  - *Hold and wait:* ✓ P2 holds R1 and R2 and waits for R3; P3 holds R3 and waits for R2.
  - *No preemption:* ✓ Resources are released voluntarily; the OS cannot forcibly reclaim them.
  - *Circular wait:* ✓ Cycle is visible.
- **Conclusion: Deadlock IS present.**

---

### RAG Example 2: Cycle Present → Deadlock IS NOT Present

**System state:**
- R1 has **2 instances**: one allocated to P3, one allocated to P2.
- R2 has **2 instances**: one allocated to P1, one allocated to P4.
- P1 requests R1.
- P3 requests R2.

```css
P1 ──requests──→ R1  (instance 1 → P3, instance 2 → P2)
P3 ──requests──→ R2  (instance 1 → P1, instance 2 → P4)
```

- A cycle is detectable algorithmically.
- **However**, P4 holds R2 but is *not* waiting for any other resource — it will complete and release R2.
  - When P4 finishes → R2's second instance is freed → P3 gets R2 → P3 completes → R1 is freed → P1 gets R1 → P1 completes.
- Similarly, P2 will eventually release its instance of R1.

**Conclusion: Cycle exists, but deadlock does NOT occur** — because multiple instances allowed an alternate execution path. This is why the rule says "maybe" and not "definitely."

---

## Methods to Handle Deadlock

Three broad approaches exist:

```css
Approach 1: Prevention / Avoidance
    → Ensure the system NEVER enters a deadlock state.

Approach 2: Detection + Recovery
    → Allow deadlock to occur, detect it, then recover.

Approach 3: Ostrich Algorithm
    → Ignore deadlock entirely.
```

### Approach 1: Prevention and Avoidance

- Design the system (via protocols or algorithms) so that the system never reaches a deadlock state.
- Sub-approach A — **Prevention:** Ensure at least one of the four necessary conditions can never hold simultaneously.
- Sub-approach B — **Avoidance:** Use advance knowledge about processes' resource needs to make allocation decisions that keep the system in a *safe state*. (Covered in Lecture 23.)

### Approach 2: Detection and Recovery

- Allow the system to potentially enter deadlock.
- Implement a separate detection algorithm that runs periodically (at fixed time intervals).
- If deadlock is detected → invoke a recovery mechanism.
- If no deadlock → wait for the next interval and run again.
- Used in systems where neither prevention nor avoidance is implemented.

### Approach 3: Ostrich Algorithm (Deadlock Ignorance)

- The OS **pretends deadlock can never happen** and does nothing about it.
- All responsibility for detecting and handling deadlock is pushed onto the **application programmer** — the OS takes no ownership.
- Named after the ostrich, which is said to bury its head in the sand when danger approaches — it "hides" from the problem.
- *Not a real solution; simply shifts the burden to the developer.*

> *We will discuss prevention and avoidance in detail, and detection + recovery in the next lecture. The Ostrich Algorithm needs no further discussion.*

---

## Deadlock Prevention

**Core idea:** If we can ensure that at least one of the four necessary conditions *never* holds, deadlock can never occur. So we design the system with protocols that make at least one condition impossible.

### Preventing Condition 1: Mutual Exclusion

- Mutual exclusion is only **necessary for non-shareable resources** — resources that genuinely cannot be used by more than one process simultaneously (e.g., a printer, a writable memory region).
- **Shareable resources** — such as a **read-only file** — can be accessed by multiple processes simultaneously because no one is writing to it, so there is no inconsistency risk. There is no need to lock them.
- **Protocol:** Use locks and mutual exclusion *only* for non-shareable resources. Do not unnecessarily make shareable resources non-shareable.
- *Limitation:* Some resources are inherently non-shareable (e.g., a printer can only handle one job at a time), so mutual exclusion cannot be completely eliminated from the system. We can only reduce it to where it is truly needed.

### Preventing Condition 2: Hold and Wait

The goal is to ensure that *whenever a process requests a resource, it is not currently holding any other resource.*

Two protocols for this:

**Protocol A — Request all resources before execution begins:**
- Before a process starts executing, it must request and be allocated *all* the resources it will ever need throughout its entire lifetime.
- It holds all resources from the start until it completes.
- *Example:* A process (a) copies a DVD to a file, (b) sorts the file (CPU work), (c) prints the file. Using Protocol A, it requests DVD drive + File + Printer at the very start, before step (a).
- *Drawback:* Resources are held idle for a long time. The printer is held from the beginning even though it's only needed at the end (step c). During the copy phase (which may take an hour), the printer sits unused — blocking other processes from using it. This is resource **underutilization**.

**Protocol B — Release all held resources before requesting new ones:**
- A process may only request additional resources when it holds *none*.
- Before requesting the next set of resources, it must release all currently held resources, then re-request everything it needs for the next phase together.
- *Example (same process):*
  - Phase 1 (copy): Request DVD + File. Perform copy. After copy completes, **release DVD and File**.
  - Phase 2 (sort): No extra resource needed (CPU handles it automatically).
  - Phase 3 (print): Request File + Printer together. Perform print. Release both.
- *Result:* The printer is only held during the print phase. During the copy phase, the printer is free for other processes.

### Preventing Condition 3: No Preemption

The goal is to allow resources to be forcibly reclaimed in certain situations, breaking the "no preemption" guarantee.

**Approach A — Implicit self-release:**
- If a process is holding some resources and requests a new resource that is *not immediately available*, then **all of its currently held resources are automatically released** (preempted back from it).
- The process is placed in a waiting state until both the previously held resources *and* the newly requested resource are simultaneously available. Then it re-acquires all of them together.
- *Sub-issue — Livelock:* When a process tries to acquire two locks simultaneously, a **livelock** can occur. Both threads keep colliding (each tries to lock at the same instant, fails, backs off, retries) — CPU cycles are wasted in a loop without either making progress. *Solution:* Insert a small sleep/delay between acquiring the two locks to break the collision pattern.

**Approach B — Targeted preemption from a waiting process:**
- If process P1 requests resource R2, and R2 is held by process P2, but P2 is itself currently *waiting* for another resource (meaning P2 is blocked, not actively using R2), then **R2 is preempted from P2 and given to P1**.
- P2 is now in a waiting state; when P1 finishes and releases R2, P2 can re-request it.
- *Example:* P1 holds R1, requests R2. P2 holds R2, requests R3. P2 has lower priority. Since P2 is waiting for R3 (not actively using R2), R2 is preempted from P2 and given to P1. P1 now has both R1 and R2, executes, then releases them. P2 can later re-request R2.

### Preventing Condition 4: Circular Wait

**Technique: Impose a fixed global ordering on all resources.**

- Assign a fixed numeric order to all resources: R1 < R2 < R3 < … < Rn.
- Enforce a protocol: *every process must request resources in strictly increasing order of their numbers.*
- A process holding Ri can only request Rj where j > i. It cannot go back and request a lower-numbered resource.

**Why this prevents circular wait:**
- Without ordering: P1 might hold R1 and request R2; P2 might hold R2 and request R1 → circular wait.
- With ordering: Both P1 and P2 must request R1 before R2. If P2 acquires R1 first, P1 must wait on R1 (not jumping ahead to R2). P2 then acquires R2 (both R1 and R2 available to it in order) and completes. R1 and R2 are freed. P1 then acquires both.
- *Circular wait cannot form* because the strictly increasing order prevents any two processes from holding and waiting in a cycle.
