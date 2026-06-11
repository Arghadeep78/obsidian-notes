## What is a Program?

- A **program** is an executable file which contains a certain set of instructions written to complete a specific job or operation on your computer.
- It is compiled code — ready to be executed.
- Stored in Disk.

## What is a Process?

- When you write code (e.g., C++) in a `.cpp` file, it is plain text — a collection of characters.
- A **compiler** takes this source file and produces an **executable** (e.g., `.exe` on Windows).
- The executable contains compiled binary/byte code — not human-readable; only the OS can interpret it.
- Executables are **platform-dependent**: different formats for Mac vs Windows.

**Example:** When you double-click MS Paint (`.exe`), the OS loads it into RAM and you can see it in Task Manager as a process named `mspaint`.

**Process = Program under execution**
- A program sitting on disk is not a process.
- When the OS brings that program from disk into RAM (main memory), it becomes a **process**.
- *The CPU can only operate on data/instructions that are in RAM — not on disk.*

---

## What is a Thread?

- A **thread** is also called a *lightweight process*.
- Every software is composed of many smaller, independent tasks.
- A **thread** is the *smallest unit of independent execution within a process* — a single sequential stream that does not depend on other parts of the process.
- Used to achieve parallelism by dividing a process's tasks which are independent paths of execution.

**Example — App syncing to cloud:**
- A user-facing app takes input, processes it, and displays output (main flow).
- Simultaneously, it needs to save that data to the cloud (background task).
- The cloud-save is independent of the main flow — it doesn't affect the user's experience.
- This background task is assigned to a **separate thread (T1)**, while the main flow continues.

**Example — JPG to PNG Converter:**

```css
Input image: 100×200 pixels
Logic handles: 100×100 at a time

Sequential approach:
  Part A (rows 0–100)  →  a1
  Part B (rows 100–200) →  b1
  Combine: a1 + b1

  Time = 10s + 10s = 20s

Multithreaded approach:
  T1 handles Part A  → a1
  T2 handles Part B  → b1
  (both run in parallel)

  Time = 10s (parallel)
```

- T1 and T2 are independent — no dependency between them.
- *Multithreading cuts execution time by parallelizing independent tasks.*

---

## When Does Multithreading Give Real Benefit?

- **Single CPU system:** Threads still context-switch, but no actual parallelism — *no performance gain.*
- **Multi-CPU / Multi-core system:** Each thread runs on a separate CPU core simultaneously — *real parallelism, real gain.*

*Multithreading is only beneficial when the hardware has more than one CPU/core.*

> A good coding practice: number of threads spawned ≈ number of available CPU cores.
> Spawning 1 lakh threads when you have 8 cores gives no added benefit — threads just context-switch on the same cores.

---

## Multitasking vs Multithreading — Key Differences

- **Multitasking:** The execution of more than one task simultaneously; concept of more than 1 process being context switched. No. of CPUs = 1.
- **Multithreading:** A process is divided into several different sub-tasks called threads, each with its own path of execution; concept of more than 1 thread being context switched. No. of CPUs >= 1 (better to have more than 1).

| Feature | Multitasking | Multithreading |
|---|---|---|
| **Unit** | Multiple processes | Multiple threads of one process |
| **Isolation & Memory Protection** | Yes — OS must allocate separate memory and resources to each program the CPU is executing | No — resources are shared among threads of that process; OS allocates memory to a process and all its threads share it |
| **Scheduling** | Processes are scheduled by OS | Threads are scheduled by OS (within a process) |
| **Context Switch Cost** | Higher — includes switching memory address space | Lower — same address space, no memory space switch |
| **CPU cache state** | Flushed on switch (different process may not use same data) | Preserved on switch (threads share same memory area) |

**Memory layout:**
```css
RAM
├── P1's memory block
│   ├── T1 (thread 1 of P1)
│   └── T2 (thread 2 of P1)
└── P2's memory block (isolated from P1)
```

- Threads within a process share the same resources allocated to that process.
- OS enforces isolation *between processes*, not between threads of the same process.

---

## Thread Scheduling

- The OS schedules threads based on **priority**.
- When spawning threads, the developer assigns priority (e.g., T1 higher priority than T2).
- The OS then allocates more CPU time slices to higher-priority threads.
- *Even though threads execute within a process, the OS assigns processor time slices to each thread individually.*

---

## Context Switching: Threads vs Processes

**Thread context switch:**
- Saves program counter, registers, and stack.
- *Does NOT switch memory address space* (threads share the same memory).
- **Faster** than process context switch.

**Process context switch:**
- Saves program counter, registers, and stack.
- *Also switches memory address space* (each process has a separate memory block).
- **Slower** — switching memory space is the heaviest part of the operation.
- CPU cache is **flushed** on process switch (old cache data belongs to the previous process).

---

## Real-World Examples of Multithreading

- **Browser tabs:** Each tab is a separate thread, allowing independent operation.

- **Text editor (e.g., MS Word):**
  - Thread 1: Handles keyboard input / actual typing
  - Thread 2: Spell-checking
  - Thread 3: Formatting
  - Thread 4: Auto-save (runs in background concurrently)

  *Without multithreading:* The editor would type → then spell-check → then format → then save — sequentially. You couldn't type again until all four tasks finished. The experience would be unusable.

---

## Summary

- **Process:** Program under execution; resides in RAM.
- **Thread:** Lightweight process; a single independent execution path within a process.
- **Multitasking:** Multiple processes sharing one (or more) CPUs via context switching and time-sharing.
- **Multithreading:** One process split into multiple threads, each executing an independent subtask. Benefit is realized only on multi-core/multi-CPU hardware.
- Threads share memory and resources of their parent process — *no isolation between them*.
- Thread context switches are faster than process context switches because memory address space is not switched.
