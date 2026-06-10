# Context Switching, Swapping, Orphan & Zombie Processes

## Recap: Short-Term and Long-Term Schedulers

```css
[Job Queue]     --LTS-->    [Ready Queue]    --STS/Dispatcher-->    [CPU]
(Secondary                  (Main Memory)
 Storage)
```

- *LTS (Long-Term Scheduler / Job Scheduler):* Picks processes from disk, places them into the Ready Queue. Also selects a healthy *mix of processes* (CPU-intensive + I/O-intensive) to prevent starvation.
- *STS (Short-Term Scheduler / CPU Scheduler):* Picks one process from the Ready Queue and dispatches it to the CPU, based on scheduling algorithm and priority.

---

## Medium-Term Scheduler (MTS) & Swapping

Swapping is necessary to improve process mix or because a change in memory requirements has overcommitted available memory, requiring memory to be freed up.

**Solution — Swapping:**
- *Swap Out:* Remove partially executed processes from the Ready Queue, save their complete state (program counter, register values, etc.) to *swap space* in secondary storage.
- *Swap In:* Once memory is freed (e.g., a memory-intensive process terminates), restore the saved state of swapped-out processes back into the Ready Queue.

```css
[Ready Queue]  ---(swap out)-->  [Swap Space / Secondary Storage]
              <--(swap in)---
```

*The MTS performs swapping.* In classical OS designs, only LTS and STS existed. MTS was introduced later to handle real-world memory pressure scenarios.

---

## Context Switching

**Real-life analogy:** You are listening to music. Your parent calls you to collect food delivery. You pause the song at the current timestamp, put down the phone (saving your current context), go downstairs, collect the food, return, and resume the music from exactly where you paused — restoring your previous context.

### What is Context Switching in OS?

When the CPU needs to stop executing one process and start executing another, it must:
1. *Save the current context* of the running process into its PCB.
2. *Load the saved context* of the next process from its PCB.

**Context = everything that describes the process's current state:**
- Program Counter (address of the next instruction to execute)
- CPU register values (all current values in every CPU register)
- Process state
- Open file descriptors (FDs)

### How it works (P1 → P2):

Say P1 was executing at instruction address 1, and the next instruction is at address 2. On a context switch:
1. The CPU saves the next instruction address (2) into P1's PCB — into the program counter field.
2. It saves all current CPU register values into P1's PCB — into the registers field.
3. It saves open file descriptors and the current process state into P1's PCB.
4. It then loads P2's saved program counter, registers, and file descriptors back into the CPU.
5. The CPU now runs P2 from exactly where P2 was last paused.

```css
CPU running P1
     |
     | (interrupt / time quantum expires / I/O request)
     v
Save P1's context --> PCB of P1
  (PC, all registers, state, FDs)
     |
Load P2's context <-- PCB of P2
  (PC, all registers, state, FDs)
     |
     v
CPU runs P2
```

- *Performed by the kernel.* The kernel contains low-level instructions (written in the kernel's code) that carry out context switching. This is not done in user space.
- *Pure overhead* — during a context switch, no useful user work is being done. From the user's perspective, the CPU is just saving and restoring state — no process from the Ready Queue is making progress. This is wasted time from the user's point of view.

### What determines context-switching speed?

- *CPU register performance* — different CPU architectures have different numbers and types of registers. The more registers, the more state needs to be saved/restored, and the longer the switch takes. Faster registers mean faster save/restore.
- *Memory speed* — the PCB is stored in RAM. DDR2 → DDR3 → DDR4 progressively increased RAM speed, which reduced context-switch time and made the overall system faster.
- *Machine-dependent* — context-switching time varies from system to system because architectures differ. There is no single universal number.

---

## Orphan Processes

### Background: How Processes Are Created

Every process is created via a `fork()` system call. A parent process calls `fork()`, which creates a child process. This means *every process in the system has a parent* — all processes form a tree structure rooted at the first OS process.

- The very first process in Linux is *`init`* — it has PID 1 and is sometimes called the "zeroth process" informally (referring to it being the root of all processes, not that its PID is 0). All other processes descend from it — the kernel's "father processes" spawn more processes, and so on down the tree.
- Because of this tree structure, the OS can track every process: it knows who created whom, and can trace any process back to `init`.

### What is an Orphan Process?

An *orphan process* is a process whose *parent has terminated before it*.

**How it happens:**
- Parent process P1 forks child process P2.
- Due to buggy code (e.g., an exception), P1 terminates before P2 finishes.
- P2 now has no parent.

**What the OS does:**
- The OS *re-parents* the orphan to the `init` process (PID 1).
- This keeps the process tree intact so the OS can continue tracking and managing all processes.

```css
Before:   init --> P1 --> P2
After P1 exits:   init --> P2  (re-parented)
```

### Live Example (Linux terminal):

A script `orphan.sh` forks a child using `sleep 200 &` — the `&` (ampersand) causes the `sleep` to run as a detached background process. Because the child is forked with `&` and the parent does not `wait()`, the parent script exits immediately after. The `sleep` process is now an orphan.

Checking with `ps -l` confirms: the `sleep` process's PPID (parent PID) has changed to `1` — the `init` process — confirming it has been adopted (re-parented) by `init`.

### Normal Process Flow (with `wait()`):

When a parent forks a child:
- The parent's responsibility is to call `wait()` and remain alive until the child finishes.
- The parent calls `wait()` to *read the child's exit status* — this tells the parent whether the child executed successfully or failed.
- Only after the parent reads the exit status does the OS remove the child's entry from the process table.
- This is the correct, clean lifecycle: parent waits → child exits → parent reads status → child entry deleted from process table → parent exits.

---

## Zombie Processes

### What is a Zombie Process?

A *zombie process* (also called a *defunct process*) is a process that has *completed execution* (exit called, all resources released) but whose *entry still remains in the process table*.

**Why does the entry remain?**
Because the parent has not yet called `wait()` to read the child's exit status. The OS keeps the entry so the parent can eventually collect the exit status.

### How a Zombie is Created:

```css
Parent (P1) forks child (P2)
     |
P2 executes --> P2 calls exit()
     |
P2's resources are freed (memory, file handles, etc.)
P2's process table entry REMAINS (zombie state)
     |
P1 eventually calls wait() --> reads P2's exit status
     |
P2's process table entry is removed (zombie reaped)
```

**The time window where P2 is a zombie:** between P2 calling `exit()` and P1 calling `wait()`.

### Why Zombies Are Problematic:

- The process table has a *finite size* (OS-defined limit).
- A zombie entry still occupies a slot in the process table.
- If many zombie processes accumulate (due to a parent that never calls `wait()`), the process table fills up.
- Once the process table is full, *no new processes can be created* — no new PIDs can be allocated.
- This is a *resource leak* (the process table itself is a resource).

### Two Problematic Scenarios:

1. **Parent never calls `wait()`:** Zombie entries accumulate indefinitely → process table exhausted → new processes can't be created.
2. **Parent itself exits before calling `wait()`:** If the parent is gone, who reaps the zombie? This represents a kernel-level bug; in practice, the `init` process may take over reaping.

### Reaping a Zombie:

*Reaping* = the act of removing a zombie's entry from the process table.
- The parent calls `wait()`, reads the exit status, and the OS removes the entry.
- This is called *"reaping of zombie processes."*

### Live Example:

A bash script forks 100 `sleep 1` processes without calling `wait()`. For a short time window, all those sleep processes appear as `Z+` (zombie) in `ps -l`. Once the parent script finishes and calls `wait()` (or the parent exits and init takes over), all zombie entries are cleared.

---

## Summary Table

| Concept | Definition | Key Point |
|---|---|---|
| Context Switch | Save current process state, load next process state | Pure overhead; kernel performs it |
| Swapping | Move partially executed process to disk and back | Performed by MTS; solves memory pressure |
| Orphan Process | Process whose parent exited before it | OS re-parents it to `init` (PID 1) |
| Zombie Process | Process that exited but entry still in process table | Parent hasn't called `wait()` yet; can exhaust process table |
| Reaping | Parent calls `wait()`, reads exit status, entry deleted | Cleans up zombie from process table |
