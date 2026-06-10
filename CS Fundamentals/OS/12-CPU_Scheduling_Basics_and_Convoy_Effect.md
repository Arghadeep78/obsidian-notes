# CPU Scheduling — Basics, Terminology & Convoy Effect

## What is CPU Scheduling?

The CPU Scheduler (Short-Term Scheduler) picks a process from the Ready Queue and dispatches it to the CPU whenever the CPU becomes idle.

```
[Ready Queue]  --STS/CPU Scheduler-->  [CPU / Running]
  p1, p2, p3                                |
     ^                                      |
     |---(time quantum / I/O done)----------+
                                            |
                                   [Exit / Terminated]
```

*Which process is picked* — that decision is made by the *CPU Scheduling Algorithm*.
*Handing over CPU control to that process* — that act is performed by the *Dispatcher*.

---

## Two Paradigms: Non-Preemptive vs Preemptive

**Non-Preemptive Scheduling**
- Once a process is given the CPU, it *holds the CPU until*:
  - It terminates, *or*
  - It voluntarily gives up CPU to do I/O (goes to Waiting state).
- Time-sharing / time quantum is *not used*.
- The process cannot be forcibly removed from the CPU mid-execution.

**Preemptive Scheduling**
- A process can be *forcibly removed* from the CPU in addition to the above two cases, when:
  - Its *time quantum expires*.
- *Time quantum* = a fixed amount of CPU time allotted to each process in time-sharing systems. After this many milliseconds/units, the process is forcibly interrupted even if it hasn't finished.
- The preempted process is placed back in the Ready Queue; another process gets the CPU.

### Comparison

| Property | Non-Preemptive | Preemptive |
|---|---|---|
| Starvation | *Higher* — no time sharing; a long job can monopolize the CPU indefinitely | *Lower* — every process gets a turn at the time quantum boundary |
| CPU Utilization | Lower — if a long job monopolizes CPU, shorter jobs can't run | *Higher* — time quantum ensures all processes get turns; more parallelism |
| Overhead | *Lower* — no context switch at every quantum; only on termination or I/O | *Higher* — context switch at every time quantum expiry (e.g., if TQ=1s, 10 context switches in 10 seconds) |

*Example for overhead:* If TQ = 1 second, then in 10 seconds there will be approximately 10 process switches → 10 context switches → 10 instances of pure overhead. Smaller TQ = more context switches = more overhead.

---

## Goals of CPU Scheduling Algorithms

When designing a CPU scheduling algorithm, the following goals (parameters) guide the design:

1. **Maximum CPU Utilization** — CPU should never sit idle.
2. **Minimum Turnaround Time** — processes should complete as quickly as possible.
3. **Minimum Waiting Time** — processes should wait as little as possible in the Ready Queue.
4. **Minimum Response Time** — a process should receive CPU for the *first time* as quickly as possible after entering the Ready Queue.
5. **Maximum Throughput** — maximize the number of processes completed per unit time.

---

## Important Terminology

**Throughput (TH)**
- Number of processes completed per unit time.

**Arrival Time (AT)**
- The time at which a process *arrives in the Ready Queue*.

**Burst Time (BT)**
- The *actual CPU time* a process needs for its complete execution — assuming it is the *only* process in the system (no waiting, no preemption).
- This is an *estimated* value in practice; the OS cannot know it perfectly in advance.

**Completion Time (CT)**
- The time at which a process *finishes execution and exits*.

**Turnaround Time (TAT)**
- Total time from when the process first entered the Ready Queue to when it terminates.
- *Formula:* `TAT = CT − AT`

**Waiting Time (WT)**
- Total time a process spends waiting in the Ready Queue (not executing, not doing I/O).
- *Formula:* `WT = TAT − BT`

**Response Time (RT)**
- Time from when the process enters the Ready Queue to when it *first receives the CPU*.
- Goal: minimize response time so users don't feel the system is unresponsive.

---

## FCFS — First Come First Served

The simplest scheduling algorithm.

- *Rule:* Whichever process arrives in the Ready Queue first gets the CPU first.
- Implemented using a simple queue (FIFO).
- *Non-preemptive.*

### Example

| Process | AT | BT |
|---|---|---|
| P1 | 0 | 20 |
| P2 | 1 | 2 |
| P3 | 2 | 2 |

**Gantt Chart:**
```
| P1 (0→20) | P2 (20→22) | P3 (22→24) |
```

| Process | CT | TAT = CT−AT | WT = TAT−BT |
|---|---|---|---|
| P1 | 20 | 20 | 0 |
| P2 | 22 | 21 | 19 |
| P3 | 24 | 23 | 20 |

**Average Waiting Time = (0 + 19 + 20) / 3 = ~13 units**

---

## Convoy Effect

The *Convoy Effect* is a major problem with FCFS.

*Definition:* If a process with a very high burst time gets the CPU first, all other processes must wait for it to finish — dramatically increasing the average waiting time.

**Analogy:** Like a slow truck on a highway holding up a long convoy of fast cars behind it.

### Why it happens:

- FCFS picks by arrival order, not burst time.
- If the first-arriving process is long (high BT), shorter processes are stuck waiting.

### Counter-example (same processes, reordered):

| Process | AT | BT |
|---|---|---|
| P2 | 0 | 2 |
| P3 | 1 | 2 |
| P1 | 2 | 20 |

**Gantt Chart:**
```
| P2 (0→2) | P3 (2→4) | P1 (4→24) |
```

| Process | CT | TAT | WT |
|---|---|---|---|
| P2 | 2 | 2 | 0 |
| P3 | 4 | 3 | 1 |
| P1 | 24 | 22 | 2 |

**Average Waiting Time = (0 + 1 + 2) / 3 = 1 unit**

*Same processes — just reordered — average waiting time dropped from 13 to 1.*

### Key insight:

The convoy effect is not limited to CPU scheduling. It applies to *any shared resource*: if one process holds a resource for a long time, everyone waiting for that resource suffers increased waiting times.

---

## Summary

- CPU Scheduling = deciding which process from the Ready Queue gets the CPU next.
- Two paradigms: *Non-preemptive* (holds until termination or I/O) and *Preemptive* (also forcibly removed at time quantum expiry).
- Five design goals: max CPU utilization, min TAT, min WT, min RT, max throughput.
- FCFS is the simplest algorithm but suffers from the *Convoy Effect* — a long job at the front makes everyone wait.
- *Convoy Effect interview question:* "What is convoy effect?" — answer: when a high-burst-time process at the front of the queue increases the average waiting time of all other processes.
