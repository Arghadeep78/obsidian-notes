## Goals of Any Operating System

Before jumping into the types, understand *why* these types evolved — they were built with specific goals in mind. Every OS type can be evaluated against these three goals:

**Goal 1 — Maximum CPU Utilization**

- The OS manages many processes that all want the CPU to run them.
- You never want the CPU to sit idle. The CPU is expensive hardware and it consumes electricity even when idle.
- Example: P1 is executing. P1 goes off to do some I/O — say, you gave a copy-paste command and it's copying from a USB drive. While that I/O is happening, the CPU has nothing to do. We do not want the CPU to just sit there doing nothing.
- The goal is: while P1 is doing I/O, put P2 on the CPU. Keep it busy. Never let it idle.

**Goal 2 — No Process Starvation**

- Imagine P1, P2, P3 are waiting to run. The OS picks P1 first.
- Now suppose P1 is a very heavy job — or worse, someone wrote a program with a `while(true)` loop and left it running infinitely.
- Because of P1 hogging the CPU, P2 and P3 never get a chance to run. They are starving.
- *Process Starvation* = a process never gets CPU time because something else is always running.
- This must not happen. Every process deserves a turn at the CPU.

**Goal 3 — High Priority Job Execution**

- Suppose P1, P2, P3 are running normally. Now suddenly a high-priority task arrives.
- Example: Antivirus software is programmed such that the moment you plug in any removable disk (USB drive), it immediately triggers a high-priority task — "scan this device now before a virus gets in."
- You want that high-priority job (call it P4) to be able to run *immediately*, displacing whatever was running.
- An OS must support this ability — to interrupt current execution and run an urgent job right away.

---

## Types of Operating Systems

There are **7 types**. The first two are historical; the next three are the most interview-critical.

---

### 1. Single Process OS

- *The simplest, most primitive type* — exactly what the name says.
- At any given time, **only one job** can execute. No multitasking at all.
- Flow: P1 executes → P1 finishes completely → only then P2 starts → P2 finishes → P3 starts.
- This was the earliest kind of OS — it came into being because the hardware at that time could only handle one process at a time. Software requirements follow hardware capabilities; you don't build software that exceeds what the hardware can support.

**Goal check:**
- *Maximum CPU Utilization?* **No.** If P1 goes to do I/O, P2 cannot step in. The CPU just sits idle waiting for P1 to come back.
- *No Process Starvation?* **Fails.** If P1 is a massive job (the `while(true)` case), P2 and P3 never get a turn. Ever.
- *High Priority Execution?* **No.** P1 is running, a high-priority job arrives — it doesn't matter, P1 will keep running. No preemption possible.

**Example:** MS-DOS [1981]

---

### 2. Batch Processing OS

- Multiple users each want to submit a job. The question is: how did they submit jobs back then?

**The Punch Card:**
- There were physical cards with rows of circles. Some circles were filled in, some were empty — this was binary information (filled = 1, empty = 0, or vice versa).
- The card was a physical paper card. You would scan it through a scanner, which converted the physical pattern of filled/empty circles into digital information.
- This is how a user submitted their program — by carrying in a punch card.
- Similar concept to how a modern ATM card or credit card has a magnetic strip with data that a scanner reads and converts to digital form, except punch cards used physical hole-punching on paper, not magnetic strips.

**How Batch Processing works:**
1. Many users walk in, each carrying their punch card (job).
2. An **Operator** (a software component) collects all the jobs.
3. The Operator cannot look *inside* each job to know how long it will take. It can only look at surface-level information — what resources each job requires.
4. Based on that surface-level view, it *sorts* the jobs in a brute-force way.
5. After sorting, the Operator groups similar jobs into **batches**.
6. Each *batch* is submitted sequentially to the CPU.
7. Within each batch, jobs execute sequentially — just like Single Process OS.

```css
Users submit jobs via punch cards
         │
   [Operator] — sorts by requirements, divides into batches
         │
   Batch 1: [J1, J4]   → submitted to CPU → J1 runs, then J4 runs
   Batch 2: [J2, J7]   → submitted to CPU → J2 runs, then J7 runs
   ...
```
![[Pasted image 20260611015631.png]]

**Goal check:**
- *Maximum CPU Utilization?* **No.** Within each batch, execution is ***still sequential***. If J1 goes to I/O, J4 just waits — CPU idles.
- *No Process Starvation?* **Fails.** If J1 within Batch 1 takes very long, J4 starves inside that batch. Additionally, if Batch 1 takes a very long time, Batch 4 may starve waiting for its turn.
- *High Priority Execution?* **No.** A batch must fully complete before the next batch runs. A high-priority job arriving mid-batch cannot interrupt.

**Example:** ATLAS OS [Manchester University, late 1950s – early 1960s]

---

### 3. Multi-Programming OS

*Important for interviews — pay close attention.*

- Still a **single CPU**, just like the previous two types.
- The key idea: maintain a **Ready Queue**.

**Ready Queue:**
- A queue data structure holding all the jobs that are ready to be executed — they just need the CPU to pick them up.
- Think of it like a waiting line: jobs sit in the queue, ready to go, waiting for their turn.
- Jobs in the Ready Queue: J1, J2, J3, J4, etc.

**How it works:**
- J1 starts executing on the CPU.
- J1 hits an I/O operation (example: needs to copy data from a USB drive). J1 goes to wait state — waiting for the I/O to complete.
- *Instead of letting the CPU idle*, the OS schedules J2 from the Ready Queue to the CPU immediately.
- J1 sits in wait state. J2 executes. When J1's I/O completes, J1 returns to the Ready Queue.
- *The CPU is almost never idle — it always has another job to switch to when the current one waits.*
- Multiprogramming increases CPU utilization by keeping multiple jobs (code and data) in **memory** so that the CPU always has one to execute in case some job gets busy with I/O.

**Goal check:**
- *Maximum CPU Utilization?* **Improved.** CPU doesn't sit idle during I/O — another job fills the gap.
- *No Process Starvation?* **Partial.** If J1 never goes to I/O and just keeps computing (no I/O trigger), J2 never gets a turn. Context switching only happens when a process voluntarily goes to wait/I/O state.
- *High Priority Execution?* **Supported.** Context switching is available here, so a high-priority job can trigger a switch.

**Context Switching** *(introduced here — important concept, covered in detail in Process Management)*

- Context Switching = the mechanism of switching the CPU from one process to another.

**Real-life analogy:**
- You are studying Physics for a board exam. Your table has all your Physics books open, your own notes, your teacher's notes, your friend's notes — everything Physics is spread out on the table.
- At 11:00 PM you planned to switch to Chemistry.
- Before switching: you put bookmarks at all the pages you were reading, close all the Physics books, stack them to the side.
- You then open all the Chemistry books, turn to the bookmarks you had already placed in them, and start studying from where you left off.
- What happened? You **saved** the current state of your Physics session (bookmarks = saved position), then **restored** the previously saved state of your Chemistry session.

**Mapping to OS:**
- CPU was working on P1. P1 goes to wait state (I/O).
- *Save P1's context into its PCB:* current memory address it was executing at, call stack contents, register values — all stored in P1's PCB.
- *Load P2's context from its PCB:* restore P2's stack, registers, address — exactly where P2 left off.
- CPU starts executing P2. P1 sits saved in its PCB until it returns from I/O.
- This save + restore process is called **Context Switching**.

**PCB — Process Control Block:**
- A small data structure that stores all information related to a process.
- Contents: current execution address, call stack, register values, process metadata.
- Every process has its own PCB.
- When context switching happens: P1's state is saved into P1's PCB; P2's state is loaded from P2's PCB.

```css
P1 executing on CPU
     │
P1 → I/O (wait state)
     │
Save P1's context → P1's PCB (address, stack, registers)
     │
Load P2's context ← P2's PCB
     │
P2 starts executing on CPU
     │
(P1's I/O completes → P1 returns to Ready Queue)
```

**Example:** THE OS [Dijkstra, early 1960s]

---

### 4. Multi-Tasking OS

- *Multi-Tasking is the logical extension (enhanced version) of Multi-Programming.*
- Still a **single CPU**. Context Switching is still used. Everything from Multi-Programming applies.
- The one major difference: **Time Sharing**.

**The Problem with Multi-Programming (that Multi-Tasking solves):**
- In Multi-Programming, context switching only happens when a process goes to wait/I/O state.
- What if a process never does any I/O? It just keeps computing — executing, executing, executing.
- That one process monopolizes the CPU indefinitely. Other processes starve even though there's no I/O to trigger a switch.

**The fix — Time Quantum:**
- Introduce a fixed time slice called a **Time Quantum**.
- Example: 100 milliseconds.
- Every process gets exactly 100ms of CPU time. When those 100ms are up, the CPU forcibly switches to the next process — regardless of whether the current process went to I/O or not.
- This is *Time Sharing*: each process gets a share of time in turns.

```css
Without time sharing (Multi-Programming):
  P1 executes ──────────────────────────── (P1 never goes to I/O, P2 starves forever)

With time sharing (Multi-Tasking):
  P1 (100ms) → P2 (100ms) → P3 (100ms) → P1 (100ms) → P2 (100ms) → ...
  Each process gets regular turns. No starvation.
```

**Goal check:**
- *Maximum CPU Utilization?* **Yes.** CPU is always busy — either doing I/O-triggered switching (like Multi-Programming) or time-quantum-triggered switching. CPU never idles.
- *No Process Starvation?* **Significantly reduced.** Even if P1 is a very heavy job, P2 still gets its 100ms quantum when P1's quantum expires. No process is locked out.
- *High Priority Execution?* **Yes.** Context switching is available. A software interrupt can immediately switch to a high-priority job. Example: an antivirus job arrives as P4 — a software interrupt fires, P4 gets the CPU immediately without waiting for a time quantum to expire.
- **Increases responsiveness** — all processes get regular CPU turns, making the system feel responsive.

*Most modern OS are Multi-Tasking OS.* When you run many apps simultaneously on your computer and they all feel responsive — that's time sharing at work.

**Example:** CTSS (Compatible Time-Sharing System) [MIT, early 1960s]

---

### 5. Multi-Processing OS

- *Multi-Processing = Multi-Tasking, but now with **more than one CPU**.*
- Context Switching: still used.
- Time Sharing: still used.
- The new addition: **multiple CPUs** (processors > 1).

**Why this is better:**
- All previous types had a single CPU that was being scheduled cleverly.
- Here, we literally have more processors doing work in parallel.
- More CPUs → more jobs running truly simultaneously → higher throughput → starvation even less likely.

**How it looks:**
```css
CPU 1: P1 (100ms) → P2 (100ms) → ...
CPU 2: P3 (100ms) → P4 (100ms) → ...

Both CPUs running simultaneously — true parallelism.
OS decides which job goes to which CPU (via process scheduling algorithms).
```

**Modern CPUs:**
- When you hear "quad-core CPU" or "octa-core CPU", that means 4 or 8 *logical processors*.
- Each core = an independent CPU for scheduling purposes.
- 8 cores = 8 separate CPUs that the OS can schedule jobs on simultaneously.

**Additional benefits:**
- **Increased Reliability:** In a single-CPU system, if that one CPU has a hardware failure — the entire system goes down. In Multi-Processing: if CPU 1 fails, CPU 2 (and CPU 3, CPU 4...) keeps running. The system doesn't crash completely. Other CPUs continue to serve jobs.
- **Better throughput** — more jobs complete per unit time due to true parallel execution.
- **Lesser process starvation** — if 1 CPU is busy with a process, others can be executed on another CPU.

**Example:** Windows NT(new tech aka modern)

---

### 6. Distributed OS

- Also called: *Loosely Coupled OS*

**Contrast with everything above:**
- Everything so far: one OS, one box (machine), one or more CPUs inside that box — all physically connected together inside the same computer.
- Distributed OS: multiple physically separate computers, each potentially with their own CPU, GPU, memory — connected to each other over a **network** (Internet or LAN).
- OS manages many bunches of resources: >=1 CPUs, >=1 memory, >=1 GPUs, etc.
- Collection of independent, networked, communicating, and physically separate computational nodes — **loosely connected autonomous** nodes.

**How it works:**
- Think of it this way: one CPU box at your house, one at your friend Babbar bhaiya's house, one more somewhere else — all interconnected over the Internet.
- These are fully autonomous, independent computers.
- A Distributed OS coordinates these separate machines so that they work together as one system.
- Jobs get distributed across the machines: P1 goes to Machine A, P2 goes to Machine B, P3 goes to Machine C, P4 goes to Machine D.
- Different machines can have different hardware configurations — one might be a low-end machine, another a high-end machine with a GPU. The OS routes work accordingly.

```css
[Machine A] ─────┐
[Machine B] ──── [Distributed OS] ─── distributes jobs across machines
[Machine C] ─────┘
       │
    Connected over Internet / LAN
```

- Multiple users can submit jobs to the distributed OS simultaneously.
- Since there are many machines (many resources), many jobs and many users can be served concurrently.

**Real-world example:**
- When you go to LeetCode, write code in the editor, and click "Submit" — your code does not run on one specific machine.
- It gets sent to a cluster of machines. Whichever machine (judge) is free picks it up, runs your code, evaluates the output, and returns the result.
- Those machines are all interconnected over the Internet. One Distributed OS coordinates all of them.
- *That infrastructure is a Distributed OS.*

**Example:** LOCUS

---

### 7. Real-Time OS (RTOS)

**Where is RTOS needed?**
- Any environment where: (1) there must be *zero or near-zero errors*, (2) execution must happen **within a strict time deadline**, and (3) computation must be extremely fast.
- *Real-time* = the system must execute within guaranteed time limits.

**Examples of where RTOS is used:**

- **Air Traffic Control (ATC):** Many flights are in the air simultaneously. ATC controllers have hardware and software that must make computations in real time. A delayed computation or an error means planes could collide. Lives depend on this being error-free and instant.
- **ROBOTS** and industrial automation systems.
- **Nuclear Plants:** Industrial control systems for nuclear reactors demand extremely high-accuracy, error-free computation that must happen quickly and within certain deadlines.
- **Industrial Applications:** Any factory automation, precision machinery, medical devices — wherever a missed deadline or an error has catastrophic consequences.

**Characteristics:**
- Error-free (or error rate minimized to the highest possible degree)
- Execution must complete within a specific, guaranteed deadline
- Very fast computation — a job given to the CPU must execute within seconds or milliseconds

*The OS that satisfies all these constraints — fast, error-free, deadline-bound — is called an RTOS.*

**Example:** ATCS

---

## OS Types at a Glance

| OS Type | CPU Count | Key Mechanism | Goal 1 (CPU Util.) | Goal 2 (No Starvation) | Goal 3 (High Priority) |
|---|---|---|---|---|---|
| Single Process | 1 | Sequential, one job at a time | ✗ | ✗ | ✗ |
| Batch Processing | 1 | Jobs grouped into batches, sequential within batch | ✗ | ✗ | ✗ |
| Multi-Programming | 1 | Context switch on I/O/wait | Improved | Partial | ✓ |
| Multi-Tasking | 1 | Context switch + Time Quantum (Time Sharing) | ✓ | ✓ | ✓ |
| Multi-Processing | > 1 | Time Sharing across multiple CPUs | ✓ | ✓ | ✓ |
| Distributed OS | Many machines | Jobs distributed across networked computers | — | — | — |
| RTOS | Varies | Error-free, deadline-bound execution | — | — | — |

---

## Interview Priority

- *All 7 names must be known.*
- **Most important for interviews: Single Process, Batch Processing, Multi-Programming, Multi-Tasking, Multi-Processing** — these five are the most frequently asked.
- The most commonly confused trio in interviews: **Multi-Programming vs Multi-Tasking vs Multi-Processing**. Keep these razor-sharp in your head.

**Quick differentiator for the confused trio:**

```css
Multi-Programming  → single CPU, context switch only on I/O/wait
Multi-Tasking      → single CPU, context switch on I/O/wait + time quantum (time sharing added)
Multi-Processing   → multiple CPUs, context switch + time sharing, true parallel execution
```

- Modern OS you use daily: **Multi-Tasking** and **Multi-Processing** — running many apps simultaneously, all responsive, no hangs.

---

## Examples of Each OS Type

| OS Type | Example |
|---|---|
| Single Process | MS-DOS [1981] |
| Batch Processing | ATLAS [Manchester University, late 1950s – early 1960s] |
| Multi-Programming | THE OS [Dijkstra, early 1960s] |
| Multi-Tasking | CTSS (Compatible Time-Sharing System) [MIT, early 1960s] |
| Multi-Processing | Windows NT |
| Distributed OS | LOCUS |
| RTOS | ATCS |
