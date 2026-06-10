# Reader Writer Problem and Solution

A classic synchronization problem frequently asked in interviews.

---

## Problem Statement

There is a **shared database** (the critical section). Two types of threads access it:

- **Reader threads** — only read data from the database.
- **Writer threads** — write (update) data to the database.

```css
Reader 1 ──┐
Reader 2 ──┤──► [ DATABASE ]
Reader N ──┘
Writer 1 ──────► [ DATABASE ]
Writer 2 ──────► [ DATABASE ]
```

---

## The Rules

**Case 1: Multiple readers reading simultaneously**
- *No issue.* Multiple readers can read at the same time — reading does not modify data, so no race condition occurs.

**Case 2: One writer writing + any other thread (reader or writer)**
- *Race condition and data inconsistency occur.*

---

## Why Simultaneous Writer Access Is Problematic

**Two writers at the same time:**
- Writer 1 writing "ABCD" and Writer 2 writing "EFGH" simultaneously → characters get interleaved → *corrupted data*.

**One writer + one reader at the same time:**
- Writer writes "ABCD", then updates to "ABZF".
- Reader reads "ABCD" before the update — *reads stale/inconsistent data*.
- The reader has no knowledge that the data changed right after the read.

---

## Goal of the Solution

- **Multiple readers** can be in the critical section simultaneously.
- **Only one writer** at a time — mutual exclusion for writers.
- *If a writer is active, no reader may enter (and vice versa).*

---

## Solution Using Semaphores

Three variables are used:

**1. `mutex` (Binary Semaphore)**
- Purpose: *Ensure `read_count` is updated with mutual exclusion.*
- `read_count` itself is shared state — multiple threads updating it can cause race conditions.
- Initial value: 1

**2. `wrt` (Binary Semaphore)**
- Purpose: *Control access to the database for both readers and writers.*
- Common to both reader and writer threads.
- Initial value: 1

**3. `read_count` (Integer, not a semaphore)**
- Purpose: *Track how many readers are currently in the critical section.*
- Initial value: 0

---

## Writer Solution

```css
Writer:
    wait(wrt)            // acquire exclusive access to database
    // → critical section: write/update data
    signal(wrt)          // release — allow others to enter
```

*Simple: only one writer at a time. `wrt` ensures mutual exclusion.*

---

## Reader Solution

```css
Reader:
    wait(mutex)              // protect read_count update
    read_count++
    if read_count == 1:
        wait(wrt)            // first reader blocks any writer
    signal(mutex)            // release read_count protection

    // → critical section: read data

    wait(mutex)              // protect read_count update
    read_count--
    if read_count == 0:
        signal(wrt)          // last reader allows writer to enter
    signal(mutex)            // release read_count protection
```

---

## How It Works — Key Logic

**First reader arriving:**
- `read_count` becomes 1.
- Calls `wait(wrt)` → if a writer is currently writing, this reader **blocks**.
- If no writer is present, `wait(wrt)` succeeds → writer is now **locked out**.
- *As long as at least one reader is present, no writer can enter.*

**Subsequent readers arriving:**
- `read_count` increments past 1.
- The `if read_count == 1` condition is false → they skip `wait(wrt)`.
- They enter the critical section directly — *multiple readers read simultaneously*.

**Last reader leaving:**
- `read_count` becomes 0.
- Calls `signal(wrt)` → *signals that writers may now enter*.
- Any writer that was blocked on `wait(wrt)` now wakes up and proceeds.

**Why `mutex` is used for `read_count`:**
- Multiple readers arriving simultaneously would all try to `read_count++`.
- `count++` is not atomic — it can cause a race condition on `read_count` itself.
- `mutex` ensures `read_count` is always updated by exactly one thread at a time.

---

## Summary of Behavior

```css
Multiple readers present → writers are blocked
No readers present → writer may enter
Writer present → all readers are blocked
Writer done → readers and next writer compete
```

---

## Important Note

- The reader solution, while correct, has a subtle fairness issue: if readers arrive continuously, a writer can be *starved* indefinitely (never gets access).
- This is the **first variant** of the reader-writer problem (reader preference).
- A second variant exists that gives preference to writers — ensuring no writer starves.

*This is a classic interview question — know both the problem setup and the semaphore-based solution well.*
