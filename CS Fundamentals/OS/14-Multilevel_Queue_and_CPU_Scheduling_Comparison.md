# Multilevel Queue Scheduling & CPU Scheduling Comparison

## Real-World vs. Simple Algorithms

The scheduling algorithms studied so far (FCFS, SJF, Priority, RR) are simplified models. Real operating systems use more sophisticated implementations that combine these algorithms. The two most important are:

1. **Multilevel Queue Scheduling (MLQ)**
2. **Multilevel Feedback Queue Scheduling (MLFQ)**

---

## Types of Processes in a Real System

Before understanding multilevel queues, recognize that processes fall into three natural categories:

**System Processes**
- Created by the OS kernel itself.
- Critical for OS operation (e.g., kernel threads, system daemons).
- Must have the *highest priority* — if OS processes don't get CPU time, the OS itself cannot function properly. You cannot deprioritize the OS's own work.

**Interactive Processes (Foreground Processes)**
- Require *user input* to proceed (e.g., Microsoft Word waiting for you to type, a text editor waiting for keystrokes).
- These processes are *waiting for I/O frequently* — every time the user presses a key or clicks, it's an I/O event. So they naturally spend a lot of time in the Waiting state and don't hog the CPU.
- Must respond quickly — if an interactive process doesn't get CPU quickly when the user acts, the application feels unresponsive and "frozen."
- Also called *foreground processes* because they are visible to the user in the GUI.

**Batch Processes (Background Processes)**
- Run in the background with *no user input* required.
- Example: a script launched with `&` in the terminal runs silently in the background — the terminal returns control immediately and the script runs without needing any user interaction.
- *No user interaction needed* → no urgency to respond quickly → lowest priority.
- Users don't notice or care exactly when a batch process finishes, as long as it eventually does.

---

## Multilevel Queue Scheduling (MLQ)

Instead of one single Ready Queue, *divide the Ready Queue into multiple sub-queues*, one per type of process.

```
            Priority
  Highest │  [System Process Queue (SP-Q)]       ← highest priority
          │       ↓
          │  [Interactive Process Queue (IP-Q)]
          │       ↓
  Lowest  │  [Batch Process Queue (BP-Q)]         ← lowest priority
```

- Each queue has its *own scheduling algorithm*.
  - Example: SP-Q → Round Robin, IP-Q → Round Robin, BP-Q → FCFS.
- *Inter-queue scheduling:* Fixed-priority preemptive scheduling between the queues themselves. If any process in a higher-priority queue is ready, it *preempts* whatever is running from a lower-priority queue.
- *A process is permanently assigned to one queue when it enters the system.* It never moves between queues.

### Scheduling Flow

- When a new process arrives, the OS assigns it to a queue based on its nature (system, interactive, or batch).
- The process stays in that queue permanently.
- *Within a queue:* scheduled by that queue's own algorithm.
- *Across queues:* higher-priority queues are always served before lower-priority ones.

```
New process → [classified by OS] → assigned to SP-Q / IP-Q / BP-Q (permanently)
                                          ↓
                              Scheduled within its queue
```

### Problem with MLQ

*The inter-queue scheduling uses fixed-priority preemptive scheduling.*
- As long as there is even one process in SP-Q, no process in IP-Q or BP-Q gets the CPU.
- Lower-priority queues (especially BP-Q) may *never* get CPU time if SP-Q and IP-Q are continuously busy — the same infinite waiting problem as priority scheduling.
- *No aging is possible* because processes cannot move between queues — a batch process stuck in BP-Q has no mechanism to ever gain priority.
- *Why no aging:* Aging requires the OS to increase a process's priority and potentially move it to a higher-priority queue. MLQ has no inter-queue movement, so aging has nowhere to move the process. The mechanism is simply absent.
- MLQ is *not commonly used today* — its inflexibility makes it a theoretically clean but practically problematic design. A more evolved version (MLFQ) replaced it.

---

## Multilevel Feedback Queue Scheduling (MLFQ)

*The improvement over MLQ. This is what modern operating systems actually use.*

**Key difference from MLQ:** Processes are *allowed to move between queues* (inter-queue movement is permitted).

### Core Idea

- Still multiple queues with different priority levels.
- A process is *not permanently bound* to a queue.
- *Separate processes based on burst time behavior over time:*
  - Processes that consume more CPU time → *demoted to lower queues*.
  - Processes that are I/O-bound or interactive (frequently yield CPU or go to Waiting) → *promoted to higher queues*.

**Why promote interactive processes?**
If a user launches an application and the OS gives more CPU time to background processes, the user's application becomes unresponsive. Since the user is actively interacting with it, it deserves a higher priority. Interactive processes frequently go to the Waiting state (waiting for keystrokes, mouse clicks), so they naturally don't hold the CPU long — they *deserve* higher priority.

### Handling Starvation (Aging in MLFQ)

- Processes that accumulate in lower queues over time get their priority gradually increased via *aging*.
- After enough time, a long-running process in a low queue is promoted back to a higher queue.
- *Aging is possible in MLFQ because inter-queue movement is allowed* (it wasn't possible in MLQ).

---

## Sample MLFQ Design

*One possible design (configurable):*

```
Queue Q0: Round Robin (TQ = 2)   ← highest priority
     ↓ (demote if not done in TQ)
Queue Q1: Round Robin (TQ = 4)
     ↓ (demote if not done in TQ)
Queue Q2: Round Robin (TQ = 8)
     ↓ (demote if not done in TQ)
Queue Q3: FCFS                   ← lowest priority
```

- A new process enters Q0.
- If it finishes within 2 units → done, exits.
- If it doesn't finish in 2 units → demoted to Q1 (gets 4 units next time).
- If it doesn't finish in 4 units → demoted to Q2 (gets 8 units).
- If it still isn't done → falls to Q3 (FCFS — run to completion).
- *Short processes finish quickly in high-priority queues.*
- *Long processes gradually move down but eventually get served.*

**Aging:** A process stuck in Q3 for too long gets its priority incremented periodically (e.g., every 15 seconds). Once priority crosses a threshold, it is moved back up to a higher queue.

---

## Five Design Parameters of MLFQ

When designing an MLFQ for a specific OS, the following must be decided:

1. **Number of queues** — how many priority levels are needed.
2. **Scheduling algorithm within each queue** — e.g., RR with what TQ, or FCFS.
3. **Method to promote a process to a higher queue** — typically via aging (priority counter increments over time).
4. **Method to demote a process to a lower queue** — typically based on TQ expiry (process didn't finish in the given TQ → demote).
5. **Which queue a new process enters initially** — typically the highest-priority queue (Q0), but can be customized based on known process type.

---

## Comprehensive Comparison of All Algorithms

| Algorithm | Design Complexity | Preemptive? | Convoy Effect | Overhead |
|---|---|---|---|---|
| **FCFS** | Simple (FIFO queue) | No | *Severe* | None |
| **SJF (Non-Preemptive)** | Complex (BT estimation required) | No | Yes | None |
| **SJF (Preemptive / SRTF)** | Complex (BT estimation required) | Yes | *None* | Yes |
| **Priority (Non-Preemptive)** | Moderate (priority assignment) | No | Yes (Infinite Waiting) | None |
| **Priority (Preemptive)** | Moderate | Yes | *Extreme* (Infinite Waiting) | Yes |
| **Round Robin** | Simple | Yes | *None* | Yes (depends on TQ) |
| **MLQ** | Complex (multi-queue setup) | Yes (between queues) | Yes (lower queues starve) | Yes |
| **MLFQ** | Most complex (configurable) | Yes | *Reduced* (aging reduces starvation; inter-queue movement redistributes CPU time) | Yes |

### Notes on Each:

**FCFS**
- No preemption → no overhead from context switching.
- The first and most illustrative algorithm to demonstrate the Convoy Effect.
- *Interview question:* "What is the preemptive version of FCFS?" → *Round Robin.*

**SJF Non-Preemptive**
- Still suffers from convoy effect if a long process arrives first (it won't be preempted).

**SJF Preemptive (SRTF)**
- Optimal average waiting time among all algorithms.
- No convoy effect because the shortest remaining job always takes priority.
- Not practical due to BT estimation difficulty.

**Priority (both versions)**
- Biggest drawback: *Infinite Waiting / Infinite Blocking* (extreme starvation).
- Solution: *Aging* — gradually increase priority of waiting processes.
- Both versions (preemptive and non-preemptive) suffer from infinite waiting.

**Round Robin**
- *Best response time* — every process gets CPU quickly.
- No starvation → no convoy effect.
- Overhead controlled by TQ: *larger TQ → less overhead, smaller TQ → more overhead.*
- Most popular algorithm; used in modern multitasking OS at various scheduling levels.

**MLQ**
- Inflexible — no inter-queue movement.
- Aging cannot be applied.
- Lower queues suffer permanent starvation.
- *Largely obsolete in practice.*

**MLFQ**
- Flexible and configurable.
- Aging applied → starvation reduced (not eliminated, but controlled).
- Used in macOS and other modern operating systems (combined with other tweaks).
- *Convoy effect is present but controlled via aging.*

---

## Key Relationships

```
FCFS  ──────────────────────────► Round Robin
(non-preemptive)          (preemptive version of FCFS)

SJF  ──────────────────────────► Special case of Priority Scheduling
(lowest BT = highest priority)

MLQ  ──────────────────────────► MLFQ
(no inter-queue movement)    (inter-queue movement + aging)
```

---

## Summary

- *Real-world OS scheduling* is not a single algorithm but a layered combination.
- MLFQ is the most practical approach — flexible, configurable, reduces starvation via aging.
- The core challenge in all scheduling: *balance between minimizing convoy effect and minimizing overhead.* Every optimization in one dimension tends to worsen the other.
- Hardware improvements (faster RAM, better CPUs) reduce context-switch overhead, making preemptive algorithms more practical over time.
