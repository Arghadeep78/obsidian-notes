# What is Concurrency

**Concurrency** is the ability of the OS to manage multiple threads of execution — either truly in parallel (on multi-core CPUs) or interleaved via context switching (on a single-core CPU). The way to achieve this is by dividing a process into *threads*.

---

## Thread

- A thread is a *lightweight process* — a sub-process or an independent path of execution within a process.
- Also called **LWP (Lightweight Process)**.
- Used to achieve *parallelism* by breaking a task into independent paths of execution.

**Definition:** A sequence of instructions within a process that can execute independently.

**Example — MS Word:**

A single MS Word process has three sub-tasks:
1. Text Editor
2. Spell Checker
3. Text Formatting

These three sub-tasks are divided into three threads — Thread 1, Thread 2, Thread 3 — each executing independently.

```css
MS Word Process
├── Thread 1: Text Editor
├── Thread 2: Spell Checker
└── Thread 3: Text Formatting
```

Without threads, these would run sequentially: type → spell check → format. With threads, all three run in parallel — the red underline appears as you type, formatting updates simultaneously.

- *This increases responsiveness.* The user sees no delay; everything feels instantaneous.

---

## Process vs Thread — Memory Layout

**Single Process:**
```css
Process P
└── Own isolated address space (independent of other processes)
```

**Threaded Process:**
```css
Process P
├── Thread T1 ──┐
├── Thread T2 ──┼── All share the SAME address space
└── Thread T3 ──┘
```

*The most important difference between a process and threads: all threads of a process share the same address space.*

---

## How Threads Get CPU Access — Thread Scheduling

- Every thread has its own **Program Counter (PC)** stored in its **TCB (Thread Control Block)** — analogous to the PCB for processes.
- Threads are scheduled by the OS using a thread scheduling algorithm.
- Threads with higher priority get more CPU access.

**Context Switching between Threads:**
- The OS saves the state of the current thread and restores the state of the next thread.
- *No address space switch happens* — both threads belong to the same process, so the same address space is used.
- Context switching between threads is therefore **much faster** than between processes.
- The CPU cache state is also preserved (same address space), so no cache reset is needed.
- Context switching between threads is triggered by I/O or TQ expiry — same triggers as process context switching.
- Thread state is saved/restored using a **TCB (Thread Control Block)**, analogous to the PCB for processes.

---

## Single CPU vs Multi-Core — Does Multi-threading Help?

**Single CPU (single core):**
- Even if a process is divided into threads, only one thread gets the CPU at a time.
- Context switching still happens between threads, so execution is effectively sequential.
- *No real parallelism benefit.*

**Multi-core CPU:**
- Thread T1 can run on CPU 1 while Thread T2 runs on CPU 2 — truly parallel.
- *This is where multi-threading gives actual benefit.*
- The advantage of multi-threading is best realized on **multi-core systems**.

---

## Memory Layout — Threads vs Process

**Process memory layout:**
```css
+----------+
|   Text   |
+----------+
|   Data   |
+----------+
|   Heap   |
+----------+
|  Stack   |
+----------+
```

**Multi-threaded process memory layout:**
```css
+----------+
|   Text   |  ← shared by all threads
+----------+
|   Data   |  ← shared by all threads
+----------+
|   Heap   |  ← shared by all threads
+----------+
| Stack T1 |  ← each thread gets its own stack
+----------+
| Stack T2 |
+----------+
| Stack TN |
+----------+
```

Each thread gets its own stack. The heap, data, and text segments are shared across all threads of the process.

---

## Benefits of Multi-threading

**1. Responsiveness**
- For interactive applications (user input + background I/O + network + formatting), each independent task can be a separate thread.
- If one thread is blocked on I/O, other threads continue executing.
- *The overall application remains responsive; no single blocking event freezes the whole process.*

**2. Resource Sharing**
- Threads of the same process share the same address space and resources.
- No IPC (Inter-Process Communication) is needed between threads — unlike between separate processes which require shared memory or message passing.
- *Thread communication is far more efficient than inter-process communication.*

**3. Economy (Cost Efficiency)**
- Context switching between threads is faster than between processes — no address space switch.
- Creating a thread is cheaper than creating a new process — no separate resource allocation required.
- *Multi-threading is more economical than multi-processing.*

**4. Scalability (Better Utilization of Multi-core CPUs)**
- On an 8-core CPU, a process can be split into 8 independent threads and assigned to 8 cores simultaneously.
- *Massive speedup for compute-intensive tasks.*

---

## Hands-on: C++ Multi-threaded Code

A process with two independent tasks (Task A and Task B) is split into two threads:

```cpp
#include <thread>

void taskA() { /* loop with sleep, prints task A output */ }
void taskB() { /* loop with sleep, prints task B output */ }

int main() {
    std::thread t1(taskA);
    std::thread t2(taskB);

    t1.join();  // main thread waits for t1 to complete
    t2.join();  // main thread waits for t2 to complete

    return 0;
}
```css

- Without `join()`: The main thread exits immediately, killing the child threads before they complete — causes a termination error.
- With `join()`: The main thread waits for both threads to finish before exiting.
- On an 8-core machine, T1 and T2 execute truly in parallel on separate cores.
- *Three threads total: main thread + T1 + T2.*
