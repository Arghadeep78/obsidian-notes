# Conditional Variable and Semaphore

---

## The Problem with Locks — Busy Waiting

When a thread is waiting for a lock to become available, it loops continuously checking the lock variable:

```
while (lock is held) {
    // spinning — checking over and over
}
```

*This wastes CPU cycles.* The thread is consuming CPU time while doing nothing useful. Since CPU cycles are extremely valuable, this is a significant inefficiency.

---

## Conditional Variable

A **Conditional Variable** is a synchronization primitive that allows a thread to **block and wait** for a signal from another thread — *without burning CPU cycles while waiting*.

It internally uses a lock (mutex) and provides two operations:
- `wait()` — the thread **blocks completely**; CPU is released and can be used by other work.
- `signal()` / `notify()` — wakes up the waiting thread; a software interrupt is sent.

*No busy waiting: the blocked thread releases the CPU entirely. It only wakes when explicitly notified.*

---

### How It Works (Conceptually)

```
Thread T1 (waiter):                   Thread T2 (signaler):
  acquire internal lock                  do work
  check condition                        call signal/notify_all
  condition is false → BLOCK ──────────► T1 wakes up
  (CPU is freed)                         T1 re-acquires lock
                                         T1 checks condition again
                                         condition is now true → proceeds
```

**Key behavior:**
- T1 was launched first, but T2 executes first.
- T1 blocks until T2 notifies it.
- *This allows threads to be synchronized in a controlled order without busy waiting.*

---

### Python Example

Two threads T1 and T2 run separate functions. T1 is launched first but is designed to wait for T2 to signal it before proceeding.

```python
import threading, time

condition = threading.Condition()  # internally wraps a mutex

def task_t1():
    with condition:
        condition.wait()            # T1 blocks here and releases the CPU
        print("T1 woke up and is running")

def task_t2():
    with condition:
        time.sleep(0.5)             # T2 does its work
        print("T2 is signaling")
        condition.notify_all()      # T2 wakes up T1
        print("T2 notification done")
```

**Output sequence:**
1. T1 is launched first but immediately blocks on `wait()`.
2. T2 runs, does its work, then calls `notify_all()`.
3. T1 wakes up and runs after T2 signals.

Even though T1 was launched first, T2 ran first — because T1 blocked on `wait()`. *This is thread synchronization without busy waiting.*

**Internally:** The `Condition` object in Python (and `std::condition_variable` in C++) wraps a mutex internally. In C++, you must supply the mutex explicitly as an argument to `std::condition_variable`.

---

## Semaphore

A **Semaphore** is an integer variable used to control access to a shared resource that has **multiple instances**.

- A mutex/lock works best for a resource with a **single instance** — only one thread enters at a time.
- A semaphore is used when a resource has **multiple instances** — several threads can access it simultaneously, up to the limit.

*Example: A large printer server with 3 sub-printers. Three threads can print simultaneously (one per printer). A 4th must wait.*

---

### How a Semaphore Works

Semaphore `S` is initialized to the number of available resource instances:

```
S = 3   (3 instances available)
```

**`wait(S)` — to acquire a resource:**
```
S = S - 1
if S < 0:
    block this thread (add to waiting queue)
```

**`signal(S)` — to release a resource:**
```
S = S + 1
if any thread is blocked:
    wake one up (send it a wakeup signal)
```

**Walkthrough with 3 instances (S=3) and 5 threads:**

```
T1 arrives: wait() → S = 2, T1 enters
T2 arrives: wait() → S = 1, T2 enters
T3 arrives: wait() → S = 0, T3 enters
T4 arrives: wait() → S = -1, T4 BLOCKED
T5 arrives: wait() → S = -2, T5 BLOCKED

T1 exits: signal() → S = -1, T4 wakes up → T4 enters
T2 exits: signal() → S = 0, T5 wakes up → T5 enters
```

*At any point, at most 3 threads are in the critical section.*

---

### No Busy Waiting in Semaphores

When a thread blocks on `wait()`, it is **properly blocked** — the CPU is freed, just like with conditional variables. No spinning on a flag.

The internal implementation looks like:
```
wait(S):
    S--
    if S < 0:
        block this thread (OS puts it to sleep)

signal(S):
    S++
    if S <= 0:
        wake up one blocked thread
```

---

### Two Types of Semaphores

| Type | Value Range | Behavior | Equivalent To |
|------|-------------|----------|---------------|
| **Binary Semaphore** | 0 or 1 | Only 1 thread at a time | Mutex / Lock |
| **Counting Semaphore** | ≥ 0 | Up to N threads simultaneously | General semaphore |

- *A binary semaphore (value = 1) behaves exactly like a mutex.*
- A mutex's internal implementation in C++ may itself be a binary semaphore.

**Python demo:**
```python
semaphore = threading.Semaphore(3)  # 3 instances
```
- With value 3: T1, T2, T3 enter simultaneously; T4 and T5 wait.
- With value 1 (binary): one thread at a time — same as a lock.
- With value 5 (= number of threads): all 5 threads enter simultaneously — no restriction.

---

### Summary Table

| Mechanism | Busy Waiting | Multiple Instances | Use Case |
|-----------|-------------|-------------------|----------|
| Lock (Mutex) | Possible (spin-lock variant) | No (1 only) | Single shared resource |
| Conditional Variable | No (blocks) | No | Order-dependent thread synchronization |
| Semaphore | No (blocks) | Yes (counting) | Resource pool with N instances |

**Note on Lock and Busy Waiting:** A simple spin-lock (like Peterson's solution or a flag-based lock) does busy wait — the thread loops checking the flag. OS-provided mutexes (Python's `threading.Lock`, C++'s `std::mutex`) are blocking — the OS puts the waiting thread to sleep instead of spinning. The conditional variable is preferred over a spin-lock because it never spins. When "lock" in this context refers to an OS-level mutex, it also does not busy wait.

*Semaphore is used when you have multiple instances of a resource and want to allow limited concurrent access.*
