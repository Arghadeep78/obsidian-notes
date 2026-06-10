# Deadlock Avoidance, Detection, and Recovery

## Deadlock Avoidance

### What Avoidance Means

- In deadlock **avoidance**, the kernel has advance information about the system state.
- The OS uses this information to decide whether to grant or delay a resource request, ensuring the system never enters deadlock.

**Information available to the OS:**
- Number of processes currently in the system
- Maximum resource need of each process (to complete execution)
- Currently allocated resources to each process
- Total number of instances of each resource type

**Goal:** Schedule processes and allocate resources in such a way that the system always remains in a **safe state**.

---

### Safe State vs. Unsafe State

**Safe State:**
> A state is *safe* if there exists a **safe sequence** — an ordering of all processes such that for each process in the sequence, the resources it still needs can be satisfied by the currently available resources plus the resources held by all processes that have already completed before it in the sequence.

- If such a sequence exists → the system can complete all processes without deadlock.
- *A safe state guarantees deadlock-free execution.*

**Unsafe State:**
> A state where no safe sequence exists — the OS cannot guarantee that all processes will complete without deadlock. *Deadlock may occur* (the four necessary conditions may become satisfied).

```
Safe State ──────────────────────────────────────────────────────────►
  ∃ safe sequence → all processes complete without deadlock

Unsafe State
  No safe sequence → deadlock may occur
```

---

### Banker's Algorithm (Deadlock Avoidance)

The **Banker's Algorithm** finds whether a safe sequence exists given the current system state.

**Tables used:**

| Column | Meaning |
|---|---|
| **Allocated** | Resources currently allocated to each process |
| **Max Need** | Maximum resources each process may ever request |
| **Remaining Need** | `Max Need − Allocated` |
| **Available** | `Total Resources − Sum of all Allocated` |

#### Worked Example

**Total resources:** A=10, B=5, C=7

| Process | Allocated (A,B,C) | Max Need (A,B,C) | Remaining Need (A,B,C) |
|---|---|---|---|
| P1 | 0,1,0 | 7,5,3 | 7,4,3 |
| P2 | 2,0,0 | 3,2,2 | 1,2,2 |
| P3 | 3,0,2 | 9,0,2 | 6,0,0 |
| P4 | 2,1,1 | 2,2,2 | 0,1,1 |
| P5 | 0,0,2 | 4,3,3 | 4,3,1 |

**Available = Total − Sum(Allocated) = (10,5,7) − (7,2,5) = 3,3,2**

**Finding the safe sequence:**

1. Available = (3,3,2). Can P1 run? P1 needs (7,4,3) — not satisfiable. Skip.
2. Can P2 run? P2 needs (1,2,2) ≤ (3,3,2). **Yes.** Schedule P2.
   - P2 completes → releases (2,0,0). Available = (3,3,2) + (2,0,0) = **(5,3,2)**
3. Can P3 run? P3 needs (6,0,0) — not satisfiable with (5,3,2). Skip.
4. Can P4 run? P4 needs (0,1,1) ≤ (5,3,2). **Yes.** Schedule P4.
   - P4 completes → releases (2,1,1). Available = (5,3,2) + (2,1,1) = **(7,4,3)**
5. Can P5 run? P5 needs (4,3,1) ≤ (7,4,3). **Yes.** Schedule P5.
   - P5 completes → releases (0,0,2). Available = (7,4,3) + (0,0,2) = **(7,4,5)**
6. Can P1 run now? P1 needs (7,4,3) ≤ (7,4,5). **Yes.** Schedule P1.
   - P1 completes → releases (0,1,0). Available = **(7,5,5)**
7. Can P3 run? P3 needs (6,0,0) ≤ (7,5,5). **Yes.** Schedule P3.
   - P3 completes. Available = **(10,5,7)** (back to total)

**Safe sequence: P2 → P4 → P5 → P1 → P3**
*The system is in a safe state.*

**Unsafe state example:** P3's remaining need for A is 6. If P3's remaining need for A were 8 instead of 6, then at the final step when P3 is the only process left to schedule, the available A would be 7 — not enough to satisfy P3's need of 8. P3 could not be scheduled. No safe sequence exists → *unsafe state (deadlock may occur)*.

**Banker's Algorithm logic:**
1. At each step, find any process whose remaining need ≤ available resources.
2. Schedule it, release its allocated resources, add them to available.
3. Repeat. If all processes are scheduled → *safe state*. If stuck with unschedulable processes → *unsafe state (deadlock may occur)*.

> *The Banker's Algorithm is the deadlock avoidance algorithm. In interviews, you are typically asked what avoidance means and what information the OS assumes — not to solve numeric examples (that is for university exams).*

---

## Deadlock Detection

**When used:** Systems where neither deadlock prevention nor avoidance is implemented.

**Approach:** Allow the system to enter deadlock. Run a detection algorithm periodically. If deadlock is detected, invoke recovery.

```
System runs
    │
    ▼
Detection algorithm runs at intervals
    │
    ├─ Deadlock detected? ──Yes──► Recovery mechanism
    │
    └─ No deadlock ──────────────► Wait for next interval, re-run
```

### Case 1: Single Instance of Each Resource Type — Wait-For Graph

- Derive the **Wait-For Graph** from the Resource Allocation Graph by:
  - Removing all resource vertices (the boxes).
  - Collapsing all edges: if P1 → R → P2 (P1 waits for a resource held by P2), this becomes P1 → P2.

```
RAG:                           Wait-For Graph:
P1 ──→ R1 ──→ P2               P1 ──→ P2
P2 ──→ R2 ──→ P1               P2 ──→ P1
```

- **If a cycle exists in the Wait-For Graph → deadlock is present.**
- **If no cycle → no deadlock.**

> *This method is valid only for single-instance resources.*

### Case 2: Multiple Instances of Each Resource Type — Banker's Algorithm

- Apply the Banker's Algorithm to the current system state.
- **If a safe sequence is found → no deadlock.**
- **If no safe sequence exists → deadlock detected.**

---

## Deadlock Recovery

Once deadlock is detected, two recovery approaches exist:

### Method 1: Process Termination

**Variant A — Abort all deadlocked processes:**
- Kill all processes involved in the deadlock simultaneously.
- *Very harsh and straightforward: guarantees deadlock removal immediately, but all involved processes lose their work.*

**Variant B — Abort one process at a time:**
- Select the process with the lowest priority.
- Kill it, freeing its held resources.
- Re-run the detection algorithm.
- Repeat until the deadlock cycle is eliminated.
- *Sounds better: avoids killing all processes if terminating just one is enough to break the cycle.*

**Example:** P1 holds R1, waits for R2. P2 holds R2, waits for R1.
- If P2 has lower priority → kill P2 → R2 is freed → P1 acquires R2 → P1 completes → deadlock resolved.

### Method 2: Resource Preemption

- Forcibly reclaim (preempt) a resource from a victim process and give it to the requesting process, until the deadlock cycle is broken.
- The victim selected for preemption must be a process that is *itself waiting for yet another resource* (i.e., it is currently blocked — not actively using all its held resources at this moment). Preempting from a process that is actively working would disrupt ongoing computation and is harder to justify.
- After preemption, the victim waits; when the requesting process finishes, the victim can re-request the preempted resource.

**Example:** P1 holds R1 and waits for R2. P2 holds R2 and waits for R3 (P2 is blocked waiting for R3, so P2 is not actively using R2 right now). P2 has lower priority.
- Preempt R2 from P2 (valid because P2 is waiting for R3, not actively using R2) → give R2 to P1 → P1 now has both R1 and R2 → P1 completes → R1 and R2 freed → P2 can eventually re-request R2.

---

## Summary: Deadlock Handling Methods

| Method | Description |
|---|---|
| **Prevention** | Eliminate at least one necessary condition (mutual exclusion, hold-and-wait, no preemption, or circular wait) by design |
| **Avoidance (Banker's)** | Use advance knowledge to schedule only safe allocations; keep system in safe state |
| **Detection + Recovery (Single instance)** | Wait-for graph; cycle → deadlock; recover via process termination or resource preemption |
| **Detection + Recovery (Multiple instances)** | Banker's Algorithm; no safe sequence → deadlock; recover similarly |
| **Ostrich Algorithm** | Ignore deadlock; delegate responsibility to application programmer |
