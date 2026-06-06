# Lecture 10 — Transactions & ACID Properties

---

## 1. What is a Transaction?

**Banking example — ABC Bank with two accounts:**
- Account **A** (yours): ₹1000
- Account **B** (friend's): ₹2000
- Task: transfer ₹50 **from A to B**.

### User perspective vs. DB perspective
- **User perspective:** a **single operation** ("send ₹50").
- **DB perspective:** NOT single — it is **six logical steps**:

```
Transaction T:
1. Read(A)            -- read A's balance
2. A = A - 50         -- temporary update in buffer
3. Write(A)           -- A becomes 950
4. Read(B)            -- read B's balance
5. B = B + 50         -- add ₹50
6. Write(B)           -- B becomes 2050
```

- These **6 logical steps in a particular sequence** = **one transaction** (≈ six CPU cycles / DB operations).

### Why it's called one transaction
- From the **user's perspective**, it's one job ("send ₹50 from my account to my friend's").
- The 6 steps must be **atomic** = **considered a single task** → either **all** steps complete (transaction completes), or if any error occurs mid-way, **don't perform the transaction** (so money never just "vanishes" from an account).

### Formal definitions
- "A unit of work done against the DB in logical steps."
- "A **logical unit of work** that contains **one or more SQL statements**." The result either:
  - **Completes successfully** → all changes made to the DB are **permanent**; or
  - **At any point of failure** → the transaction is **rolled back**.

### Roll back — why
- If A is debited (₹1000 → ₹950) and then an error hits while reading B (server error, hardware failure, logical error), the **debited ₹50 is credited back** (rolled back).
- Sometimes the rollback isn't instant (can take 24h / days), but it happens.
- **Reason:** maintain **consistency** — **money can neither be destroyed nor created.** Total in the system must stay ₹3000. (If ₹50 is debited but not delivered and not rolled back → ₹2950 total → ₹50 destroyed → not allowed.)

---

## 2. Read / Write / Commit Operations

### Read(A)
- The DB lives on disk/storage. The running program works in **main memory (RAM)** via a **local buffer**.
- `Read(A)` copies A's value from the DB into the local buffer (DB still shows 1000; buffer now has 1000).

### Write(A)
- Takes the modified value from the **local buffer** and transfers it to the **actual DB storage (disk/files)**.

### Commit
- In **real (advanced) DB systems**, writes don't go directly to the physical DB; the write also stays in a **RAM buffer** temporarily.
- The final **`Commit`** operation is when changes are **actually persisted** to the physical DB.
- SQL has a `COMMIT` command. Transaction commands include **`BEGIN TRANSACTION`** and **`END TRANSACTION`**.

```
  Read / Write / Commit — data flow between layers:

  [Disk / DB Storage]  <----Write----  [RAM Buffer]  <----Read----  [Program / CPU]

  Step 1, Read(A):  DB disk --copy A=1000--> RAM buffer  (disk still shows 1000)
  Step 2, A=A-50:   RAM buffer: A becomes 950            (disk still shows 1000)
  Step 3, Write(A): RAM buffer --950--> DB disk           (disk now shows 950)

  In advanced systems: Write puts the value into a WRITE BUFFER in RAM first.
  Commit is the operation that flushes the write buffer to actual disk storage.
  This is why "Commit" is a separate concept from "Write".
```

> For understanding in this lecture: treat `Write` as writing to the physical DB.

---

## 3. ACID Properties

> Purpose: **to ensure the integrity of data.** Every transaction MUST follow these four properties, else the DB becomes **inconsistent**.

The transaction used throughout: the 6-step A→B transfer above.

---

### 3.1 Atomicity
- **Either ALL operations of the transaction are reflected properly in the DB, or NONE.**
- All 6 steps complete → reflected in DB → transaction successful. Any failure mid-way → don't perform the transaction.
- Things are **digital (0 or 1), not fuzzy** — the transaction either fully happens or doesn't.

**Example of violation:**
- A: 1000 → after `Read(A)`, `A - 50` = 950, `Write(A)` → A = 950.
- If the **system crashes** right after A becomes 950 (before B is credited):
  - A = 950, B = 2000 → total = 2950 → **₹50 lost** → **inconsistent state.**

**How atomicity recovers:**
- The DB must either complete the transaction **or recover to the old state** (rollback).
- The DB **maintains the old state** (and intermediate state). On failure before commit, **roll back to the old state**:
  - Old state: A = 1000, B = 2000 (total ₹3000) → DB stays **consistent**.

```
  Atomicity — only two valid outcomes:

  SUCCESS path (all 6 steps execute without error):
  Step1 -> Step2 -> Step3 -> Step4 -> Step5 -> Step6 -> COMMIT
  DB state: A=950, B=2050. Total=3000. Consistent.

  FAILURE path (crash after Step 3, before Step 4):
  Step1 -> Step2 -> Step3 -> CRASH
                               |
                    At this point: DB shows A=950 (already written),
                    B=2000 (not yet updated). Total=2950. Inconsistent!
                               |
                               v  ROLLBACK
                    Undo Step3: restore A to 1000.
                    DB state: A=1000, B=2000. Total=3000. Consistent again.

  The DB maintains the OLD STATE values so it can undo partial writes.
  Without atomicity, the ₹50 would be "destroyed" (A debited, B never credited).
```

---

### 3.2 Consistency
- The **integrity constraints must be maintained before AND after** the transaction.
- The **sum** in the whole banking system stays the same: ₹3000 before = ₹3000 after (debit from one = credit to another).
- "Money could not be created or destroyed."
- **If atomicity is implemented correctly, consistency is maintained.** (The properties are interrelated.)

```
  Consistency — the invariant that must hold before and after:

  Before transaction: A=1000, B=2000  --> Total = 3000
  After transaction:  A=950,  B=2050  --> Total = 3000  <-- CONSISTENT

  The integrity constraint here is: sum(all accounts) = constant.
  Any other integrity constraint (e.g., balance >= 0, data types, FKs)
  must also hold before AND after the transaction.

  Connection to Atomicity: 
    If atomicity fails (partial execution left behind), consistency breaks.
    If atomicity is correctly enforced (full commit or full rollback),
    consistency is automatically preserved — they work together.
```

---

### 3.3 Isolation
- The system may have many concurrent transactions: `T1, T2, T3, ... Tn`.
- Multiple transactions can run **concurrently**, but each must stay **isolated** — and the system **guarantees the result is as if they ran sequentially**.

**Concurrency problem example (without isolation):**
- `T1` (GPay) and `T2` (Net Banking) both start near the same time and both `Read(A) = 1000`.
- `T1` computes 950, writes A = 950. `T2` (read 1000 earlier) also computes 950, writes A = 950.
- Both transfer ₹50 to B → B gets ₹50 twice → B = 2100, but A = 950.
- Result: **extra ₹100 introduced** into the system → not allowed.

```
  Isolation violation — the "dirty read" / race condition problem:

  Timeline (no isolation, T1 and T2 run concurrently):

  T1: Read(A)=1000   A-50=950   Write(A)=950   Read(B)=2000   B+50=2050   Write(B)=2050
  T2:   Read(A)=1000   A-50=950   Write(A)=950   Read(B)=2050              B+50=2100  Write(B)=2100

  What went wrong:
    T2 reads A=1000 BEFORE T1 has written A=950.
    Both T1 and T2 subtract 50 from 1000, so A ends up at 950 (only one debit)
    but B gets credited TWICE (2000+50+50 = 2100).
    End state: A=950, B=2100. Total = 3050 -> ₹50 created out of nothing!

  With Isolation (transactions appear sequential):

  T1 runs to completion first:
  T1: Read(A)=1000  A=950  Write(A)=950  Read(B)=2000  B=2050  Write(B)=2050  COMMIT

  T2 runs after T1 commits:
  T2: Read(A)=950   A=900  Write(A)=900  Read(B)=2050  B=2100  Write(B)=2100  COMMIT

  End state: A=900, B=2100. Total = 3000. Consistent.
  (Both ₹50 transfers happened correctly, one after the other.)
```

**Fix — run transactions sequentially (logically):**
- Run `T1` first; until `T1` **terminates** (commits or fails), don't start `T2`.

**Whole-system view + OS callback:** The above example used only our own account. Looking at the **whole banking system**, there is some `N` amount of money that must stay **consistent**. When you put debit/credit requests against this total money — *recall from OS (Operating Systems): the minus (debit) operation should be an **atomic operation**, so that another **thread** or transaction doesn't come in and read its old (stale) value.*

**Formal guarantee:** Even though multiple transactions execute concurrently, the system guarantees that for **every pair of transactions `Ti` and `Tj`**, it appears to `Ti` that **either**:
- `Tj` **finished execution before `Ti` started**, **or**
- `Tj` **started execution after `Ti` finished** (sequential).

> Locks are implemented so that things **appear logically sequential**, keeping the total-money resource consistent. In other words: **multiple transactions happen in isolation** — Transaction A doesn't know Transaction B is happening; they execute independently and don't interfere.

---

### 3.4 Durability
- **After a transaction completes successfully, its changes must be made persistent in the DB** — even if a system failure occurs afterward.
- Problem to avoid: you tell the user "transaction complete," but due to an **end-state error / system failure**, the change never **persisted** to the DB.

**Example:**
- The 6 steps are done in main memory (buffer): A = 950, B = 2050. You declared the transaction complete.
- Now the buffered values are being **written to disk**, and the **system crashes** at that point.
- Main memory gets flushed (lost) on crash → the change might be lost.
- The DB must have the **capability to recover**: on restart, **re-generate** this state and re-apply the write so the DB reflects A = 950, B = 2050.

```
  Durability — the problem and the solution:

  Problem scenario:
    Transaction executes all 6 steps in RAM buffer.
    DB declares: "Transaction complete!" User is notified.
    Now the buffer writes are being flushed to disk...
    SYSTEM CRASHES at this point.
    RAM is volatile -> buffer contents are LOST.
    Disk may still show A=1000, B=2000 (old state).
    But the user was told the transfer succeeded. Contradiction!

  Solution — the DB's Recovery Management component:
    Before or during each step, the DB writes a LOG ENTRY to stable storage:
      "Step 3: Write(A), new value = 950"
      "Step 6: Write(B), new value = 2050"
    On restart after crash: read the log -> REDO the writes to disk.
    Disk is updated to A=950, B=2050 -> correct, durable state restored.

  Other mechanisms: write to disk intermediately, checkpoints.
  All serve the same purpose: ensure committed changes survive crashes.
```

**Mechanisms (durability ensured by the Recovery Management component):**
- **Generate logs** of each operation as the transaction runs (e.g., "−50", "Write(A)", "Read", ...) and store them. On restart/recovery, read the logs and **redo** the operations into the actual DB.
- Or **write to disk intermediately** instead of using only main memory.
- Or use **checkpoints**.

> Durability is important because we told the user the transaction is complete; if the back-end/server DB then fails, we must be able to **reconstruct what happened** and re-apply it to the actual DB. (Covered further later.)

---

## 4. States of a Transaction (Life Cycle)

```
                         (read/write ok)
   [Active] ───────────────────────────► [Partially Committed] ──(commit)──► [Committed]
       │                                          │                              │
       │ (error during read/write)                │ (failure)                    │
       ▼                                          ▼                              ▼
    [Failed] ◄────────────────────────────────────┘                       [Terminated]
       │ (rollback / undo)
       ▼
    [Aborted] ──► back to State 1 (prior state) ──► [Terminated]
```

### 4.1 Active State
- The **very first state** of the life cycle.
- **Read and Write operations** happen here.
- If they execute **without any error** → move to **Partially Committed**.
- If an error occurs **during a read** (e.g., reading A = 1000 fails) → go directly to **Failed**.

### 4.2 Partially Committed State
- "After the transaction is executed, the changes are **saved in the buffer in main memory**."
- All work happens in main memory (DB is a storage device on disk/SSD/server like S3; CPU does the ± work in main memory, so data is loaded there).
- From here:
  - If changes are **made permanent in the DB** → move to **Committed**.
  - If there is **any failure** (e.g., during `Write(A)`) → go directly to **Failed**.

> Atomicity is enforced by **checking each step**: if a step isn't complete, go straight to Failed (don't introduce inconsistency).

### 4.3 Committed State
- "When the updates are made permanent in the DB, the transaction is **committed**." **Rollback cannot be done** after commit.
- Think of the DB moving from **State 1 (old values) → State 2 (new permanent values)**: A = 950, B = 2050.
- Reaching here = **transaction successful.**

### 4.4 Failed State
- "When a transaction is being executed and **some failure** occurs" — hardware, software, communication issue, interrupt, etc. — and it's **impossible to continue** execution → go to **Failed**.
- After Failed, the second duty is **recovery**: undo whatever changes were made so far → this is **Roll Back**.

### 4.5 Aborted State
- "When a transaction reaches the Failed state, **all changes made in the buffer are reversed** (undone)." After roll back, the transaction reaches **Aborted**.
- After rollback/abort and passing through **Terminated**, the DB returns to **State 1 (the state prior to the transaction)** — because State 2 was never reached (failure occurred mid-way).

### 4.6 Terminated State
- A transaction is **terminated** if it is **either Committed or Aborted** — i.e., it ended (successfully or not).
- In SQL: between **`BEGIN TRANSACTION`** and **`END TRANSACTION`**, the partially-committed work happens; on a problem → Failed → roll back to a **checkpoint / state**; exiting the block → **Terminated** (terminated via either abort or commit).

```
  Transaction Life Cycle — reading the state diagram:

  Every transaction starts at Active.
  From Active, two exits:
    (a) Read/Write executed without error --> Partially Committed
    (b) Error during a Read/Write         --> Failed

  From Partially Committed, two exits:
    (a) Successfully flushed to disk (Commit) --> Committed --> Terminated
    (b) Failure during the final write         --> Failed

  From Failed, one exit:
    Rollback all buffer changes --> Aborted --> Terminated (back to old DB state)

  Committed is the ONLY success exit. Aborted and the Terminated
  reached via Aborted both restore State 1 (the DB before the transaction).

  Key point: Rollback is NOT possible from Committed state.
  Once changes are permanent, they stay permanent (Durability).
```

---

## 5. Summary

- A **transaction** = a logical unit of work (one or more SQL statements / multiple logical steps) that must be **atomic** — all-or-nothing.
- **ACID:**
  - **Atomicity** — all operations reflected, or none (rollback on failure).
  - **Consistency** — integrity constraints hold before and after; total resource preserved.
  - **Isolation** — concurrent transactions appear sequential; no interference.
  - **Durability** — committed changes persist even after system failure (via logs / disk writes / checkpoints; handled by Recovery Management).
- **Transaction states:** Active → Partially Committed → Committed (success); or → Failed → Aborted → Terminated (rollback to prior state).

**Next lecture (11):** How to implement Atomicity.
