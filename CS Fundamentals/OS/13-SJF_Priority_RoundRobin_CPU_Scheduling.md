# SJF, Priority Scheduling & Round Robin

## SJF — Shortest Job First

*Rule:* Among all processes currently in the Ready Queue, the one with the *lowest burst time* gets the CPU.

- Motivation: Resolves the Convoy Effect seen in FCFS — instead of serving the first-arrived process, serve the shortest one first to minimize overall waiting time.

*Key problem:* Burst time cannot be known perfectly in advance. The OS must *estimate* it — using heuristics such as: the historical execution time of previous runs of the same job, the code/program size, or other patterns. This estimation is done before scheduling. However, this is a *near-impossible task* in practice: a process might have a `while(true)` loop that makes its actual burst time infinite even though the estimator predicted it to be small. The estimate can be completely wrong, and SJF acts entirely on this estimate.

---

### SJF Non-Preemptive

- Once a process gets the CPU, it *runs to completion* (or until I/O).
- No preemption even if a shorter job arrives mid-execution.

**Selection criterion:** Arrival Time + lowest BT among currently available processes.

#### Example

| Process | AT | BT |
|---|---|---|
| P1 | 0 | 8 |
| P2 | 1 | 4 |
| P3 | 2 | 9 |
| P4 | 3 | 5 |

**Gantt Chart:**
```
| P1 (0→8) | P2 (8→12) | P4 (12→17) | P3 (17→26) |
```

*(At t=8, all processes have arrived. Among P2, P3, P4 → P2 has BT=4, P4 has BT=5, P3 has BT=9 → P2 goes next. At t=12, remaining are P3 and P4 → P4 has BT=5 → P4 goes.)*

| Process | CT | TAT = CT−AT | WT = TAT−BT |
|---|---|---|---|
| P1 | 8 | 8 | 0 |
| P2 | 12 | 11 | 7 |
| P3 | 26 | 24 | 15 |
| P4 | 17 | 14 | 9 |

**Average Waiting Time = (0 + 7 + 15 + 9) / 4 = 7.75 units**

*Does convoy effect occur in SJF Non-Preemptive?*
- Yes — if the first process (at t=0) has a very high BT, it runs uninterrupted, and shorter jobs that arrive later must wait.

---

### SJF Preemptive (also called SRTF — Shortest Remaining Time First)

- *At any point a new process arrives*, compare its BT to the *remaining* BT of the currently running process.
- If the new process has a *lower BT*, the current process is *preempted* and the new one gets the CPU.

**Selection criterion:** Arrival Time + lowest *remaining* BT at any given moment.

#### Example (same processes as above)

**Gantt Chart construction:**

- t=0: Only P1 available → P1 starts. Remaining BT = 8.
- t=1: P2 arrives (BT=4). P1 remaining = 7. Since 4 < 7 → *P1 preempted*, P2 scheduled.
- t=2: P3 arrives (BT=9). P2 remaining = 3. Since 3 < 9 → P2 continues.
- t=3: P4 arrives (BT=5). P2 remaining = 2. Since 2 < 5 → P2 continues.
- t=5: P2 finishes. Remaining: P1(7), P3(9), P4(5). Lowest = P4(5) → P4 scheduled.
- t=10: P4 finishes. Remaining: P1(7), P3(9). Lowest = P1(7) → P1 scheduled.
- t=17: P1 finishes. Only P3 left → P3 runs.
- t=26: P3 finishes.

```
| P1(0→1) | P2(1→5) | P4(5→10) | P1(10→17) | P3(17→26) |
```

| Process | CT | TAT | WT |
|---|---|---|---|
| P1 | 17 | 17 | 9 |
| P2 | 5 | 4 | 0 |
| P3 | 26 | 24 | 15 |
| P4 | 10 | 7 | 2 |

**Average Waiting Time = (9 + 0 + 15 + 2) / 4 = 6.5 units**

*Does convoy effect occur in SJF Preemptive?*
- *No.* Even if a long job arrives first, as soon as any shorter job arrives, the long job is preempted. It continuously gets pushed down. The average waiting time is *always the lowest* among all scheduling algorithms — *SJF Preemptive gives the optimal (minimum) average waiting time.*

*Core limitation of SJF:*
- Estimating burst time is *near-impossible* in practice. A process with a while(true) loop would be estimated as short but runs indefinitely.

---

## Priority Scheduling

*Rule:* Each process is assigned a priority. The process with the *highest priority* gets the CPU.

- *SJF is a special case of Priority Scheduling* — where priority is determined by burst time (lowest BT = highest priority). The difference is that here, priority is *explicitly assigned* rather than derived from BT.

---

### Priority Scheduling Non-Preemptive

- Once a process gets the CPU, it runs until termination or I/O.

#### Example

| Process | AT | BT | Priority |
|---|---|---|---|
| P1 | 0 | 4 | 2 |
| P2 | 1 | 2 | 6 |
| P3 | 2 | 3 | 3 |
| P4 | 3 | 5 | 5 |
| P5 | 4 | 1 | 4 |
| P6 | 5 | 4 | 12 |
| P7 | 6 | 6 | 9 |

*(Higher number = higher priority in this example)*

**Gantt Chart (step-by-step):**
- t=0: Only P1 in Ready Queue → P1 runs. Non-preemptive, so it runs all 4 units.
- t=4: P1 done. P6 has not yet arrived (arrives at t=5). Available: P2(priority=6), P3(priority=3), P4(priority=5), P5(priority=4). Highest = P2(6) → P2 runs 2 units.
- t=6: P2 done. Now P6 has arrived (at t=5). Available: P3(3), P4(5), P5(4), P6(12), P7 not yet. Highest = P6(12) → P6 runs 4 units.
- t=10: P6 done. P7 arrived at t=6. Available: P3(3), P4(5), P5(4), P7(9). Highest = P7(9) → P7 runs 6 units.
- t=16: P7 done. Available: P3(3), P4(5), P5(4). Highest = P4(5) → P4 runs 5 units.
- t=21: P4 done. Available: P3(3), P5(4). Highest = P5(4) → P5 runs 1 unit.
- t=22: P5 done. Remaining: P3(3) → P3 runs 3 units.
- t=25: P3 done. All processes complete.

*Execution order: P1 → P2 → P6 → P7 → P4 → P5 → P3.*

| Process | CT | TAT = CT−AT | WT = TAT−BT |
|---|---|---|---|
| P1 | 4 | 4 | 0 |
| P2 | 6 | 5 | 3 |
| P3 | 25 | 23 | 20 |
| P4 | 21 | 18 | 13 |
| P5 | 22 | 18 | 17 |
| P6 | 10 | 5 | 1 |
| P7 | 16 | 10 | 4 |

**Average Waiting Time = (0+3+20+13+17+1+4) / 7 ≈ 8.3 units**

*Convoy effect:* Present. P3 (lowest priority) waits 20 units despite having a small BT of 3. Low-priority jobs suffer severely.

---

### Priority Scheduling Preemptive

- Whenever a *higher-priority* process arrives, the currently running process is preempted immediately.
- The algorithm checks the Ready Queue at every time unit (or at every new arrival) and asks: is there now a process with higher priority than the one currently running?

#### Example (same processes)

**Step-by-step Gantt Chart:**
- t=0: Only P1 (priority=2) available → P1 runs.
- t=1: P2 arrives (priority=6). 6 > 2 → P1 preempted (P1 has 3 BT remaining), P2 runs.
- t=2: P3 arrives (priority=3). 3 < 6 → P2 continues.
- t=3: P2 completes (BT=2 fully used). Available: P1(BT=3,pr=2), P3(BT=3,pr=3), P4(BT=5,pr=5). Highest = P4(5) → P4 runs.
- t=4: P5 arrives (priority=4). 4 < 5 → P4 continues. P4 has now used 1 unit of its 5 BT (4 remaining).
- t=5: P6 arrives (priority=12). 12 > 5 → P4 preempted. P4 ran from t=3 to t=5 (2 units used, *3 BT remaining*). P6 runs.
- t=6: P7 arrives (priority=9). 9 < 12 → P6 continues.
- t=9: P6 completes (BT=4 used). Available: P1(BT=3,pr=2), P3(BT=3,pr=3), P4(BT=3 remaining,pr=5), P5(BT=1,pr=4), P7(BT=6,pr=9). Highest = P7(9) → P7 runs.
- t=15: P7 completes (ran 6 units). Available: P1(BT=3,pr=2), P3(BT=3,pr=3), P4(BT=3 remaining,pr=5), P5(BT=1,pr=4). Highest = P4(5) → P4 runs its remaining 3 units.
- t=18: P4 completes. Highest remaining = P5(4) → P5 runs 1 unit.
- t=19: P5 completes. Highest remaining = P3(3) → P3 runs 3 units.
- t=22: P3 completes. Only P1 remains (priority=2, BT=3) → P1 runs.
- t=25: P1 completes. All processes done.

```
| P1(0→1) | P2(1→3) | P4(3→5) | P6(5→9) | P7(9→15) | P4(15→18) | P5(18→19) | P3(19→22) | P1(22→25) |
```

*P2 completed at t=3. P4 ran 2 units (t=3→5), was preempted, resumed at t=15 for its remaining 3 units. Every segment boundary = one context switch = overhead.*

| Process | CT | TAT = CT−AT | WT = TAT−BT |
|---|---|---|---|
| P1 | 25 | 25 | 21 |
| P2 | 3 | 2 | 0 |
| P3 | 22 | 20 | 17 |
| P4 | 18 | 15 | 10 |
| P5 | 19 | 15 | 14 |
| P6 | 9 | 4 | 0 |
| P7 | 15 | 9 | 3 |

**Average Waiting Time = (21+0+17+10+14+0+3) / 7 ≈ 9.3 units**

*Observation:* The sheer number of segments in the chart directly reflects how often context switches occur in the real system — every bar boundary is pure overhead. *The preemptive version has far more overhead than the non-preemptive version.*

*Convoy effect:* *Extremely severe.* P1 (the first process to arrive, priority=2) had to wait until t=22 before getting CPU again — a waiting time of 21 units. Even though P1 arrived first, it ran for only 1 unit at the start, then was preempted and starved by every higher-priority process that arrived. A low-priority job waits regardless of how short or how early it arrived — priority alone determines order, not burst time.

---

### Infinite Waiting (Starvation) — The Biggest Drawback of Priority Scheduling

*Infinite Waiting (Infinite Blocking):* In priority scheduling, low-priority processes may *never* receive the CPU if high-priority processes keep arriving. This is the *extreme version of the Convoy Effect*.

**Famous example (rumor, also cited in textbooks):**
> At MIT, an IBM 7094 system was turned on in *1967*. Several jobs were submitted. Low-priority jobs were supposed to run "eventually." When the system was checked in *1973*, those low-priority jobs were still sitting in the Ready Queue — they had never received CPU time in 6 years.

---

### Solution to Infinite Waiting: Aging

*Aging:* Gradually increase the priority of processes that have been waiting for a long time.

- Example: Every 15 seconds (configurable), increment the priority of all waiting processes by 1.
- After enough time, even the lowest-priority process will accumulate enough priority to be scheduled.
- Prevents any process from waiting indefinitely.

---

## Round Robin (RR)

*The most popular and widely used CPU scheduling algorithm. Used in multitasking operating systems today.*

*Rule:* Each process gets the CPU for a fixed *time quantum (TQ)*. After TQ expires, it is preempted and placed at the back of the Ready Queue. The next process in line gets the CPU.

- *No explicit priority or BT estimation needed.* Criterion = Arrival Time + Time Quantum.
- *FCFS with preemption* — processes are served in arrival order but only for one time quantum at a time.
- *Designed for time-sharing systems.*
- *Easy to implement* — maintain a circular queue; give each process TQ time.

**Properties:**
- *Lowest starvation* — every process gets CPU within at most (n−1) × TQ time where n = number of processes.
- *No Convoy Effect* — no single process can hog the CPU beyond one TQ.
- *Overhead is higher* — frequent context switches at every TQ.
- *Response time is the best* — every process gets CPU quickly at least once.

### Overhead and Time Quantum

- If TQ is *large* → fewer context switches → lower overhead → but approaches FCFS behavior.
- If TQ is *small* → more context switches → higher overhead → but better response time.
- *Overhead in RR is determined by the time quantum size.*

### Example

| Process | AT | BT |
|---|---|---|
| P1 | 0 | 4 |
| P2 | 1 | 3 |
| P3 | 2 | 2 |
| P4 | 3 | 1 |
| P5 | 4 | 6 |
| P6 | 6 | 3 |

*Time Quantum = 2*

**Queue tracking and Gantt Chart:**

```
t=0:  Queue=[P1]. Run P1 for 2. P1 remaining=2.
t=2:  Queue=[P2,P3,P1]. Run P2 for 2. P2 remaining=1.
t=4:  Queue=[P3,P1,P4,P5,P2]. Run P3 for 2. P3 done (BT=2). 
t=6:  Queue=[P1,P4,P5,P2,P6]. Run P1 for 2. P1 done (BT=2). 
t=8:  Queue=[P4,P5,P2,P6]. Run P4 for 1. P4 done (BT=1). 
t=9:  Queue=[P5,P2,P6]. Run P5 for 2. P5 remaining=4.
t=11: Queue=[P2,P6,P5]. Run P2 for 1. P2 done (BT=1).
t=12: Queue=[P6,P5]. Run P6 for 2. P6 remaining=1.
t=14: Queue=[P5,P6]. Run P5 for 2. P5 remaining=2.
t=16: Queue=[P6,P5]. Run P6 for 1. P6 done.
t=17: Queue=[P5]. Run P5 for 2. P5 done.
t=19: All done.
```

*(Actual completion order and times depend on exact queue state at each step.)*

*Observation:* The Gantt chart for RR is significantly more complex to construct due to frequent preemptions — this directly reflects the high overhead in the real system.

---

## Implementation Hint

- **SJF** → can be implemented using a *min-heap* on burst time. As processes arrive, insert into heap; always pop the minimum BT process.
- **Round Robin** → implement with a *circular queue*.
- **Priority Scheduling** → implement with a *priority queue*.

Implementing these algorithms in code (using DSA knowledge) is strongly recommended to solidify understanding.

---

## Summary Comparison

| Algorithm | Preemptive? | Convoy Effect | Overhead | Notes |
|---|---|---|---|---|
| FCFS | No | Severe | None | Simplest; first-come basis |
| SJF (Non-Preemptive) | No | Yes | None | BT estimation required |
| SJF (Preemptive / SRTF) | Yes | None | Yes | Optimal avg WT; BT estimation required |
| Priority (Non-Preemptive) | No | Yes (Infinite Waiting) | None | Fixed priority; starvation risk |
| Priority (Preemptive) | Yes | Yes (Extreme) | Yes | Aging solves starvation |
| Round Robin | Yes | None | Yes (depends on TQ) | Best response time; most popular |
