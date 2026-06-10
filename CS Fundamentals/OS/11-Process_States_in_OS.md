# Process States in OS

## What is a Process State?

Every process, throughout its entire lifecycle — from generation (new state) to termination — passes through several distinct states. The PCB (Process Control Block) stores the *current state* of the process at all times.

---

## The Five Process States

**New State**
- When the OS is in the process of converting a program into a process.
- During this conversion, the process state stored in the PCB is *New*.
- The process is *being created* — it does not yet fully exist.

**Ready State**
- The process has been created and loaded into memory.
- It is sitting in the *Ready Queue*, waiting for CPU allocation.
- It is ready to execute — the only thing it needs is the CPU.
- Multiple processes can be in the ready queue simultaneously (this is what enables multiprogramming).

**Running State**
- The CPU has been allocated to the process.
- The process is actively executing.
- The PCB for this process will show: state = *Running*.

**Waiting State (Blocked State)**
- The process is waiting for an I/O operation to complete.
- It cannot proceed until the I/O is done.
- It does *not* go directly back to Running after I/O — it goes back to the *Ready Queue* first, then waits to be scheduled again.

**Terminated State**
- The process has finished execution.
- It no longer exists in memory.
- All resources held by the process are released.

---

## Process Lifecycle Diagram

```
[DISK]
  |  LTS picks process
  v
[NEW STATE]
  |  admitted to memory
  v
[READY QUEUE] <-----(1)---- [RUNNING STATE: time quantum expires / preempted]
     |                              |
     | CPU Scheduler                |---(2)---> [TERMINATED]  (execution complete)
     | + Dispatcher                 |
     v                              |---(3)---> [WAITING / BLOCKED]  (I/O request)
[RUNNING STATE]                                      |
                                                     | I/O completes
                                                     v
                                              [READY QUEUE]  <-- back here, NOT Running

(1) = preemption or time quantum expiry sends process back to Ready Queue
(2) = process finishes execution
(3) = process requests I/O; must go to Waiting first, then Ready after I/O done
```

- *A process coming back from Waiting goes to the Ready Queue, not directly back to Running.*

---

## Types of Queues

**Job Queue**
- Contains all processes in the *New State*.
- Held in secondary storage (disk).
- The *Job Scheduler (Long-Term Scheduler / LTS)* picks processes from here and loads them into the Ready Queue.

**Ready Queue**
- Contains all processes in the *Ready State*, residing in main memory.
- Managed by the *CPU Scheduler (Short-Term Scheduler / STS)*.
- The degree of multiprogramming = number of processes that can reside in the Ready Queue at once.

**Waiting Queue (Device Queue)**
- Contains processes waiting for I/O completion.
- Multiple processes can be waiting simultaneously (e.g., P2 and P3 both doing I/O while P1 runs).

```
[Job Queue]     [Ready Queue]     [CPU / Running]
(Secondary  --> (Main Memory) --> (Executes)
 Storage)
    ^                                   |
    |                              [Waiting Queue]
    |                                   |
    +------------ (back to Ready) ------+
```

---

## Types of Schedulers

**Job Scheduler (Long-Term Scheduler — LTS)**
- Picks processes from a *large pool of programs* on disk (secondary storage) and loads them into the Ready Queue.
- An important responsibility of the LTS is to select a *mix of processes* — a balanced combination of CPU-intensive and I/O-intensive jobs. If only CPU-intensive jobs are loaded, the I/O devices stay idle; if only I/O-intensive jobs are loaded, the CPU stays idle. Either extreme wastes resources and increases starvation risk.
- *Controls the Degree of Multiprogramming* — since it decides which processes get loaded from disk into the Ready Queue, it directly controls how many processes can be in the Ready Queue at once.
- *Runs infrequently* — its idle time between activations is long (on the order of minutes). Example: at t=0 it loads five jobs into the Ready Queue; at t=0+1 minute it checks again whether more jobs need to be loaded. This long gap is why it is called the *long-term* scheduler.

**CPU Scheduler (Short-Term Scheduler — STS)**
- Picks a process from the Ready Queue and dispatches it to the CPU, depending on the scheduling algorithm and priority.
- *Runs very frequently* — its idle time between activations is extremely small (milliseconds). The moment a process goes to I/O or its time quantum expires, the STS immediately checks the Ready Queue for the next process to dispatch.
- Example of why low frequency is bad: if P1 was scheduled and then went for I/O after a few instruction cycles, and the STS then waited one full minute before checking the Ready Queue again, the CPU would sit idle for that entire minute — which is unacceptable. The STS is therefore designed with near-zero idle time.
- *Short-term* because of this extremely short interval between activations.

**Medium-Term Scheduler (MTS)**
- *Introduced in modern systems* (not present in classical OS designs).
- Handles *swapping*.
- When memory is under pressure (too many memory-intensive processes in the Ready Queue), it *swaps out* some processes to secondary storage (swap space).
- When memory frees up, it *swaps in* those processes back to the Ready Queue, restoring their saved state.

---

## Swapping

- **Swap Out:** A partially executed process is removed from the Ready Queue and saved to secondary storage (SSD/HDD) — its exact state (program counter, registers) is preserved.
- **Swap In:** The saved process is later loaded back into memory and restored to the Ready Queue.
- *Swapping is performed by the Medium-Term Scheduler (MTS).*

```
[Ready Queue]  --swap out-->  [Swap Space / Secondary Storage]
              <--swap in---
```

**Why swapping is needed:** If many memory-intensive processes are loaded simultaneously, the available RAM becomes exhausted and the Ready Queue can no longer be maintained. Swapping temporarily removes some processes to free memory.

**Real-world analogy:** On Android/iOS, apps you opened a while ago seem to still be "open" but reload when you tap them — because they were swapped out to storage.

---

## Dispatcher

- The *Dispatcher* is the OS module that actually *gives CPU control to the selected process*.
- It performs the context switch, switches to user mode, and jumps to the correct location in the program.
- The CPU Scheduler *decides which process* gets the CPU; the Dispatcher *carries out* that handover — it enables the selected process to actually run.
- The term "dispatch" is used because the Dispatcher is dispatching (sending/assigning) the CPU to the chosen process.
- *Note:* The Dispatcher is distinct from the CPU Scheduler. The scheduler decides; the dispatcher acts.

---

## Summary

| Concept | Description |
|---|---|
| New | Process being created |
| Ready | In memory, waiting for CPU |
| Running | CPU allocated, executing |
| Waiting | Blocked, waiting for I/O |
| Terminated | Execution finished |
| Job Queue | Processes in New state (on disk) |
| Ready Queue | Processes in Ready state (in RAM) |
| Waiting Queue | Processes waiting for I/O |
| LTS | Loads processes into Ready Queue; controls degree of multiprogramming |
| STS | Dispatches process from Ready Queue to CPU; high frequency |
| MTS | Handles swapping; manages memory pressure |
