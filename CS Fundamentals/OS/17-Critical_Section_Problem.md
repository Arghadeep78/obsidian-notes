# Critical Section Problem

---

## How `count++` Actually Works at the CPU Level

When you write `count++` in code, at the machine instruction level it is **not atomic** — it involves at least two distinct CPU-level operations:

```
temp = count + 1      ← read current value from memory, add 1, store result in a CPU register (temp)
count = temp          ← write the register value back to memory
```

*`temp` here is not a named variable in code — it represents a CPU register. Because there are two separate memory operations, a context switch can occur between them.*

---

## Shared Resource & the Problem

Consider an Aadhaar center analogy:
- A central database (shared resource) tracks a citizen count.
- Multiple branch offices (threads A, B, C, D) all send `count++` requests to the database simultaneously.

```
Thread A ──┐
Thread B ──┼──► Database (Shared Resource) ← count++
Thread C ──┤
Thread D ──┘
```

Because `count++` involves two separate memory operations (read-then-write), a **context switch can happen between those two steps**:

**Example — count = 11, Thread A and Thread B both send count++:**

```
Thread A: temp = 11 + 1 = 12   ← context switch happens here!
Thread B: temp = 11 + 1 = 12   ← reads old count (still 11)
Thread A: count = temp = 12
Thread B: count = temp = 12
Final count = 12  ← WRONG, should be 13
```

Two increment requests were made but only one took effect. *One update is silently lost.*

---

## Race Condition

This problem — where the final result depends on the order in which threads are scheduled — is called a **Race Condition**.

- *Race Condition occurs when two or more threads access shared data and try to change it simultaneously.*
- The result of the data change is **dependent on the scheduling algorithm** — which thread gets the CPU first.
- The output is **non-deterministic** and **inconsistent**.

A Python demo confirms this: running two threads each incrementing a counter 1 million times should yield 2 million, but the actual output is inconsistent and changes on every run.

---

## Critical Section

**Critical Section** — a segment of code where:
- A *shared resource* is accessed (shared variable, file, database, etc.)
- A **write operation** is being performed on it
- Multiple threads access it concurrently

*The name "Critical" reflects how important it is to handle this section carefully — incorrect access leads to data inconsistency.*

---

## Three Conditions Any Critical Section Solution Must Satisfy

**1. Mutual Exclusion**
- At any point in time, *only one thread can be inside the critical section*.

**2. Progress**
- If no thread is currently in the critical section, *any thread that wants to enter should be free to do so*.
- There must be no fixed ordering that prevents a thread from entering.
- Any thread should be able to make progress into the critical section.

**3. Bounded Waiting**
- No thread should wait *infinitely* to enter the critical section.
- There must be a time bound after which a waiting thread is guaranteed to enter.
- *Note: Conditions 1 and 2 are mandatory. Condition 3 is important but less strictly enforced.*

---

## Solutions to the Critical Section Problem

### Solution 1: Atomic Operation
- Make `count++` a single atomic operation — one that completes in a single CPU cycle without interruption.
- C++ supports `atomic` variables: `std::atomic<int> count;`
- An atomic variable is **thread-safe**: any manipulation completes fully before a context switch occurs.
- *Limitation: not available in all languages.*

### Solution 2: Mutual Exclusion using Locks (Mutexes)

A **lock** acts like a room key — only one person enters at a time. The next person must wait outside until the first exits.

```
Thread T1 → acquire lock → critical section → release lock
Thread T2 → tries to acquire → BLOCKED until T1 releases → enters
```

Python demo with a lock applied to the `count++` critical section always produces the correct, consistent output of 2 million — regardless of how many times the program runs.

### Solution 3: Semaphores
*(Discussed separately in next lecture)*

---

## Single Flag Solution (Attempt)

**Proposed:** Use a `turn` variable (0 or 1) to decide which thread enters the critical section.

```
turn = 0 → T1 enters critical section, T2 waits
turn = 1 → T2 enters critical section, T1 waits
```

**Analysis:**
- *Mutual Exclusion: ✓ achieved* — only one thread enters at a time.
- *Progress: ✗ NOT achieved* — the order is fixed by the initial value of `turn`. If `turn = 0`, T1 always goes first. There is no freedom for either thread to enter when the critical section is free.

*Single flag solution is incomplete — it violates the Progress condition.*

---

## Peterson's Solution

An improvement over the single flag solution. Uses **two variables**:

- `flag[2]` — an array of booleans; `flag[i] = true` means thread i is *ready to enter* the critical section.
- `turn` — indicates *whose turn* it is.

**How it works (for two threads T1 and T2):**

```
T1:
  flag[0] = true
  turn = 1
  while (flag[1] == true && turn == 1) { wait }
  // critical section
  flag[0] = false

T2:
  flag[1] = true
  turn = 0
  while (flag[0] == true && turn == 0) { wait }
  // critical section
  flag[1] = false
```

**Analysis:**
- *Mutual Exclusion: ✓* — only one thread is in the critical section at a time.
- *Progress: ✓* — there is no fixed sequence; whichever thread the CPU schedules first will enter. Both T1 and T2 have equal opportunity.

**Limitation:** *Peterson's solution works only for two threads.* It does not extend to more than two threads.

---

## Disadvantages of Locks

**1. Contention / Busy Waiting**
- If T1 holds the lock and T2 is waiting, T2 spins in a while loop checking the lock — **wasting CPU cycles**.
- If T1 crashes or throws an exception while holding the lock, T2 (and T3, etc.) wait **indefinitely** — because the lock is never released.
- *Write careful code: ensure a lock is always released, even on exceptions.*

**2. Deadlock**
- Situation: P1 holds R1 and needs R2. P2 holds R2 and needs R1.
- Neither can proceed. Neither releases. *The system is permanently stuck.*

```
P1 holds R1, waiting for R2
P2 holds R2, waiting for R1
→ Deadlock
```

**3. Difficult Debugging**
- Multi-threaded code is hard to debug because threads run concurrently — execution order is non-deterministic.
- Stepping through with a debugger can shift context unpredictably between threads.

**4. Starvation**
- A low-priority thread acquires the lock on the critical section.
- A high-priority thread arrives but must wait for the low-priority thread to release.
- *The high-priority thread starves even though it has higher importance.*
