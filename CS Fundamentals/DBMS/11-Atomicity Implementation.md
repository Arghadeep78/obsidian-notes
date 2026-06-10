## 0. Recap

The two ACID properties implemented here:

- **Atomicity** — A transaction's operations are atomic: the transaction either **completes fully** or **does not complete at all**; it never stops in the middle. (e.g., money cannot be deducted and then "lost" / not returned). If a transaction **fails, it is rolled back.**
- **Durability** — Once a transaction is **completed and the user is notified** ("transaction has been completed"), all the updates made inside the DB **must persist.** It must not happen that the user was told "complete" but the DB does not reflect the changes.

This covers **how to implement atomicity and durability.**

- Atomicity = "all or nothing."
- Durability = "once done, stays done — even after a failure."

---

## 1. Shadow Copy Scheme

The simplest / most naive way to implement atomicity and durability.

### Concept & Logic

Core components (both on **disk**):

- **Old DB copy (before update)** — the existing database copy on disk before any update operation.
- **DB Pointer** — also stored **on disk**. At **any instant of time**, this pointer points to the **current DB copy**.

> **Why the pointer determines "current truth":** The DBMS reads the DB Pointer on every startup to find where the current database lives on disk. Whoever the pointer points to is the DB the system uses. This is why flipping the pointer is the atomic commit step — before the flip, the old copy is "current"; after, the new copy is.

**Mechanics (step by step):**

1. When a transaction wants to update, it first makes a **complete copy** of the database in **RAM**.
2. **All operations are performed on the new copy** (in RAM).
3. When the transaction completes, all pages of the new copy are **written back to disk** (new copy on disk).
4. **Even after writing, the transaction is NOT yet committed.** We do not say "transaction successful" yet.
5. The transaction is said to be **committed only when the DB Pointer (on disk) is updated to point to the new disk location.**
6. Once the DB pointer is updated to the new location, **then** we notify the user that the transaction has committed (change is now permanent).

> **Critical point:** The transaction *is said to have committed* exactly at the point where the updated DB pointer on disk is switched from the old location to the new location.

```css
BEFORE TRANSACTION:
  ┌─────────────────────────────────────────────────────┐
  │  DISK                                               │
  │                                                     │
  │  DB Pointer ──────────────► Old DB Copy (addr 0x01) │
  │  (holds 0x01)               (A=1000, B=1000)        │
  └─────────────────────────────────────────────────────┘

DURING TRANSACTION (steps 1–3):
  ┌──────────────────────────┐   ┌──────────────────────────┐
  │  DISK                    │   │  RAM                     │
  │                          │   │                          │
  │  DB Pointer ──► Old DB   │   │  New Copy (being built)  │
  │  (still 0x01)   Copy     │   │  (A=950, B=2050)         │
  │                 (A=1000, │   │                          │
  │                  B=1000) │   │  All writes happen here  │
  │                 UNTOUCHED│   │  (old copy is safe)      │
  └──────────────────────────┘   └──────────────────────────┘
  Step 3: When complete, write new copy's pages from RAM → disk
  (new copy now exists on disk too, but pointer hasn't moved yet)
  → Transaction is still NOT committed.

AFTER COMMIT (step 5 — DB Pointer flips atomically):
  ┌─────────────────────────────────────────────────────┐
  │  DISK                                               │
  │                                                     │
  │  DB Pointer ──────────────► New DB Copy (addr 0x02) │
  │  (now holds 0x02)           (A=950, B=2050)         │
  │                                                     │
  │  Old DB Copy (addr 0x01) ← now unreferenced,        │
  │  (A=1000, B=1000)           can be deleted          │
  └─────────────────────────────────────────────────────┘
  ✅ COMMIT = the instant the pointer write completes (0x01 → 0x02)
  Only now do we notify the user: "transaction successful."
```

### Key Assumption / Constraint

- Only **one transaction at a time** — transactions must run **serially**, not concurrently. The scheme works only if transactions are **serialized**.

### How Atomicity is Maintained

- **If the transaction aborts** (user clicks abort) **at any point before the DB pointer is updated:**
  - The system simply **deletes the new copy** from RAM (calls free).
  - The **old copy is unaffected**, and the **DB pointer still points to the old copy** (it never moved).
  - Result: **either all updates are reflected, or none** → this is atomicity.
- Transaction abort = just delete the new copy.

```css
ABORT SCENARIO (at any point before the pointer flips):

  DISK:
  DB Pointer ──► Old DB Copy         New Copy in RAM ──► [free()/deleted]
  (still 0x01)   (A=1000, B=1000)    (A=950, B=2050)         [discarded]
                  ^ untouched

  Because the DB Pointer was never moved (still 0x01),
  the old copy IS the current database.
  Deleting the new copy from RAM leaves the DB in its original state.

  Result: Either all updates are reflected (pointer moved) or none (pointer stayed).
  → Atomicity achieved.
```

### How Durability is Maintained

The transaction is considered successful/committed **only when the DB pointer is updated**.

- **Case A — System fails BEFORE the updated pointer is written to disk:**
  - New copy may be written, but the pointer still holds the **old** value (e.g., `0x01`, not `0x02`).
  - On **restart**, the system **reads the DB pointer**, finds the old value, and sees the **original (old) copy**.
  - The transaction is treated as **not committed** → durability is fine because it was never committed.

- **Case B — System fails AFTER the DB pointer has been updated** (e.g., `0x02` written):
  - On **restart**, the pointer points to the **new copy**, all pages of the new copy were already written to disk before failure.
  - The committed data **persists** → durability maintained.

```css
DURABILITY CASES:

  Case A — Crash BEFORE pointer flip (0x01 never changed to 0x02):
  ┌──────────────────────────────────────────────────────────────┐
  │ DB Pointer = 0x01  Old DB Copy (0x01)  New DB Copy (0x02)   │
  │ (unchanged)         A=1000, B=1000      A=950, B=2050        │
  │                     ↑ still "current"   (orphaned on disk)   │
  │ CRASH happened here                                          │
  │ Restart → system reads DB Pointer → finds 0x01              │
  │         → uses Old DB Copy as the current DB                 │
  │ Transaction is treated as NOT committed (pointer never moved)│
  │ → Durability is fine: user was never told "committed" ✅     │
  └──────────────────────────────────────────────────────────────┘

  Case B — Crash AFTER pointer flip (0x01 already changed to 0x02):
  ┌──────────────────────────────────────────────────────────────┐
  │ DB Pointer = 0x02  New DB Copy (0x02) fully on disk          │
  │ (updated)           A=950, B=2050 ← all pages written        │
  │                     before pointer flipped                   │
  │ CRASH happened here (after pointer flip)                     │
  │ Restart → system reads DB Pointer → finds 0x02              │
  │         → uses New DB Copy as the current DB                 │
  │ All committed data is there (new copy was fully on disk)     │
  │ → Durability maintained ✅                                    │
  └──────────────────────────────────────────────────────────────┘

  NOTE: Step 3 (write new copy to disk) happens BEFORE step 5 (flip pointer).
  This ordering is what makes Case B work — the new copy is always fully
  on disk before the pointer ever moves.
```

### The "Atomic Pointer Write" Question

> **Q:** What if the system fails *while* the DB pointer itself is being written (changing `0x01 → 0x02`)?

- The whole implementation **depends on the write to the DB pointer being atomic.** This single write must be atomic.
- **Hardware support:** Disk systems provide **atomic updates for a single block / disk sector** — a sector write either fully happens or does not happen.
- **Therefore:** we make sure the **DB pointer lies entirely within a single sector**, by storing the DB pointer at the **beginning of a block** (single sector).
- So `0x01 → 0x02` either fully happens or does not — it can never fail in the middle.

> The pointer write is atomic because the pointer is stored within a single disk sector, and hardware guarantees sector writes are atomic.

### Drawback

- **Inefficient.** Every transaction creates a **complete copy** of the DB. If the DB is huge (terabytes), copying it every time is expensive.

### Where it lives

- Implemented inside the **Recovery Mechanism Component** of the DBMS. (Shadow copy and the schemes that follow live here.)

---

## 2. Log-Based Recovery System

A **better** approach than shadow copy — **no need to make a full copy** of the DB.

### Concept & Logic

- While a transaction is executing, we simultaneously **maintain logs.**
  - On transaction start → write a log.
  - On each value change inside the transaction → write a log.
- These logs are stored in **stable storage** (not in the same storage as the DB).

### Stable Storage

- A storage that **guarantees atomicity** — its write operations either fully happen or do not (atomic writes).
- It is **robust**: it can **recover from power failure and hardware failure.**
- **Why a separate, expensive storage?** If logs were kept in the **same storage as the DB**, then when the DB storage fails, the **logs would vanish too** — defeating the purpose. So logs go to a **separate, expensive, stable storage.**

```css
ARCHITECTURE:

  ┌──────────────────┐       ┌────────────────────────────┐
  │   APPLICATION    │       │   STABLE STORAGE           │
  │   (RAM / DBMS)   │──────►│   (separate, robust disk)  │
  │                  │ logs  │   LOG FILE                 │
  └────────┬─────────┘ first │   <T0, start>              │
           │                 │   <T0, A, 950>             │
           │ actual writes   │   <T0, B, 2050>            │
           ▼ only after log  │   <T0, commit>             │
  ┌──────────────────┐       └────────────────────────────┘
  │   DATABASE       │
  │   (main disk)    │
  │   A = 950        │  ← updated only after log is written to stable storage
  │   B = 2050       │
  └──────────────────┘

  KEY: Stable storage is SEPARATE from the DB disk.
       If the DB disk dies, logs survive. If both somehow die,
       that's a hardware catastrophe beyond what the scheme handles.
       Logs go to stable storage FIRST, then DB is updated.
```

### Example Setup (used throughout)

Accounts:
- **A** = 1000 (initial)
- **B** = 1000 (initial)

Transaction T0 (send 50 from A to B):
```css
read(A)
A = A - 50
write(A)
read(B)
B = B + 50
write(B)
```

### Log Record Format

As the transaction runs, logs are generated. A log record contains the **transaction identifier**, the **variable** changed, and the **new value**:

```css
<T0, start>
<T0, A, 950>     # T0 wrote new value 950 to variable A  (1000 - 50)
<T0, B, 1050>    # T0 wrote new value 1050 to variable B (1000 + 50)
<T0, commit>
```

> **Note on values:** Both A and B start at 1000. T0 transfers 50: A → 950, B → 1050.

### Write-Ahead Rule (very important)

- The log must be stored *BEFORE* the actual database operation happens.
- You **first write the log** (what you intend to do), **then perform** the actual operation on the DB.
- **Why:** If you wrote the log *after* the actual operation, on failure you wouldn't know what was happening. Recording intent first lets you **recover** correctly afterward.

```css
WRITE-AHEAD RULE (for every operation):

  ✅ CORRECT ORDER:
  Step 1: Write log to stable storage  →  "<T0, A, 950>"
  Step 2: Perform actual DB write      →  A = 950 in DB

  ❌ WRONG ORDER:
  Step 1: Perform actual DB write      →  A = 950 in DB
  Step 2: Write log                    →  CRASH here = no log = cannot recover!
```

> **Write-ahead logging rule:** Log first, then write to DB. This ensures that on any crash, there is always a record of what was happening.

---

## 3. Deferred Database Modification

The **first type / implementation method** of log-based recovery.

### Concept & Logic

- **Ensure atomicity by recording all DB modifications in the log,** but **defer (delay) the execution of all write operations until the transaction has been fully executed (committed).**
- "Deferred" = **postpone / do later.** Modifications to the actual DB are done **later**, not while the transaction is running.

**Flow:**

1. Start the transaction → write `<T0, start>`.
2. Write the logs for each intended change (e.g., `<T0, A, 950>`, `<T0, B, 1050>`) — **but the actual DB is NOT modified yet.**
3. When the transaction completes → write `<T0, commit>`.
4. **Only after `commit` is written**, use the logs to **actually execute** the modifications on the real DB (e.g., set A's balance to 950, B's to 1050).

> Up to commit, you were **only writing logs**. After commit, you go to the DB and apply the deferred modifications.

```css
DEFERRED MODIFICATION TIMELINE:

  T0 running:
  ──────────────────────────────────────────────────────────►  time
  │            │            │             │           │
  start       log A=950    log B=1050   commit     apply writes
  <T0,start>  <T0,A,950>  <T0,B,1050>  <T0,commit>  A←950, B←1050
                                                     (actual DB)
  [────────── only logs written to stable storage ──────][DB updated]
                DB is UNTOUCHED during this phase

  KEY INSIGHT: The DB is only modified AFTER <T0, commit> is safely
               written to stable storage. If crash happens anywhere
               in the left phase, no DB cleanup needed — nothing was
               written to the actual DB yet.
```

### How Atomicity is Maintained

- While the transaction runs, **no actual DB modification happens** — only logs are written.
- **If the system crashes / transaction is aborted BEFORE `commit`:**
  - On inspection, there is **no `commit` record** → the transaction never committed.
  - **Simply ignore those logs** — nothing else to do.
  - The DB write logs are ignored because the transaction failed.
- Result: **either all updates are reflected, or none** → atomicity.

**Summary:** If system crashes before transaction complete (or user aborts), the information written in the logs is **simply ignored** — nothing else.

```css
CRASH BEFORE COMMIT:
  Stable Storage: <T0,start>, <T0,A,950>, <T0,B,1050>   ← no <commit> record!
  DB State:       A=1000, B=1000                         ← untouched (deferred)

  Recovery: Scan logs → no <T0,commit> found
            → T0 is treated as never committed
            → Ignore all T0 log entries — do nothing to the DB
  DB remains: A=1000, B=1000 (exactly as before T0 started)

  Atomicity: The DB was never touched (deferred), so nothing to undo.
             Result is as if T0 never ran → perfectly atomic ✅
```

### Durability (Deferred Modification)

- Once the transaction is **complete**, the records associated with it in the log file are executed → write `<T0, commit>`.
- After commit, the **deferred writing** begins (write 950, write 2050 into the actual DB).

> Durability if the system crashes **after** `commit` is written but **before/during** the deferred writes are applied: because the committed log records exist in stable storage, on restart the system **redoes** the writes from the log to ensure persistence.

```css
CRASH AFTER COMMIT (during deferred writes):
  Stable Storage: <T0,start>, <T0,A,950>, <T0,B,1050>, <T0,commit>
                                                         ↑ commit exists!
  DB State (partial — crash happened mid-write):
    A = 950  (written)    B = 1000 (not yet written — crash happened here)

  On Restart:
  → Scan stable storage → find <T0, commit>
  → REDO: apply all logged values → write B = 1050 into actual DB
  → Both A and B are now consistent ✅
  Durability: Because commit was logged in stable storage before crash,
              we can always redo the writes on restart.
```

---

## 4. Immediate Database Modification


The **second type / implementation method** of log-based recovery — the opposite of deferred.
### Concept & Logic

- DB modifications are **output to the actual DB while the transaction is still in the active state** (i.e., the real DB is updated *during* the transaction, not delayed until after commit).
- These DB modifications written by an **active (not-yet-committed)** transaction are called **uncommitted modifications.**
- The write-ahead rule still applies: an **update takes place only after its log record has been written to stable storage** (log first, then DB).

> **Difference from Deferred:** In deferred, DB is unchanged until commit. In immediate, DB is changed live during the transaction — which means if you abort, you must undo what you already wrote.

### Log Record Format (needs the old value too)

- Because the real DB is changed before commit, recovery may need to **undo** those changes. So each log record stores **both the old value and the new value** of the changed variable:

```css
<T0, start>
<T0, A, 1000, 950>     # variable A: old value 1000, new value 950
<T0, B, 1000, 1050>    # variable B: old value 1000, new value 1050
<T0, commit>
```

### Failure Handling (uses old value & new value fields)

In the event of a **crash or transaction failure**, the system uses the log records to restore correctness:
**Failure handling:**
- **System failure / transaction abort BEFORE commit:**
  - Use the **old-value field** to **UNDO** the modification (restore A back to 1000, B back to 1000 — the values before T0 started).
  - → **Atomicity** maintained.
- **Transaction complete (commit written) then system crash:**
  - On restart, the log has `commit` → use the **new-value field** to **REDO** the transaction (replace old field with new field).
  - Only redo transactions that **have a `commit` log record**; if no `commit`, assume the transaction failed.
  - → **Durability** maintained.
- **Golden rule:** Update takes place **only after the log record is in stable storage** (write log first, then actual update — else recovery is impossible).

```css
IMMEDIATE MODIFICATION — RECOVERY DECISION TREE:
  System restarts after crash
           │
           ▼
  Is <T0, commit> in the log?
     │                       │
    YES                      NO
     │                       │
     ▼                       ▼
  REDO                     UNDO
  (use NEW values           (use OLD values from log
   from log to restore       to reverse any writes
   committed state)          already made to the DB)
  → Durability ✅           → Atomicity ✅

  Example (T0: A 1000→950, B 1000→1050, transferring 50 from A to B):
  Log had commit  → REDO: write A=950, B=1050 into DB
  Log had NO commit → UNDO: write A=1000 back, B=1000 back into DB
```

> **Deferred vs Immediate:** Deferred only ever needs **REDO** (DB untouched before commit, so nothing to undo). Immediate may need **UNDO** *and* **REDO** because the DB is modified while the transaction is still active — hence each log record must keep the **old value** as well.

---
## 5. Checkpoint Log Based Recovery Method
 Logs keep accumulating as thousands/lakhs of transactions run → **stable storage fills up** and **cost increases**. Real-time systems can't store unlimited old information.
- A **checkpoint** is basically an **announcement.** At a point where the **DB state is consistent** (all prior transactions are in **commit** state), create a checkpoint (e.g., `C1`, `C2`).
- A specific `<checkpoint>` marker is appended directly into the active transaction log file on stable storage (disk).
- If a later transaction fails, you **recover to that checkpoint position** — you don't need to scan all logs from the beginning.
- It builds on the base methods above.

```css
CHECKPOINT CONCEPT:

  Log timeline:
  ──────────────────────────────────────────────────────────►
  T1,T2,T3      [C1]      T4,T5,T6      [C2]      T7,T8 ← CRASH
  (committed)            (committed)              (active)

  Without checkpoints: On crash, scan ALL logs from T1 onwards.
  With checkpoints:    On crash, only scan logs from [C2] onwards.
                       T1–T6 already confirmed consistent at C2.

  Benefit: Less log scanning → faster recovery → lower storage cost.
```

## Summary Table

> **Log-based recovery is better than shadow copy.** If the system fails between the `0x01` and `0x02` pointer write: the disk system performs **atomic operations on a particular block/sector**, so the DB pointer update is atomic.

| Scheme | Idea | Atomicity | Durability | Cost |
|---|---|---|---|---|
| **Shadow Copy** | Copy whole DB in RAM, work on copy, swap an atomic DB pointer on disk | Abort = delete new copy; pointer never moved | Commit = pointer updated atomically (single sector, HW-atomic) | **Inefficient** (full copy each time); transactions must be **serial** |
| **Log-Based (Deferred Modification)** | Write logs (`<T0,A,950>`, `<T0,B,1050>`) to stable storage first; apply DB writes only after `commit` | No `commit` ⇒ ignore logs; DB was never touched | Logs in stable storage survive failure → REDO on restart | No full copy needed |
| **Log-Based (Immediate Modification)** | Write logs (with **old + new** values) first, then apply DB writes **during** the active transaction (uncommitted modifications) | Crash before commit ⇒ **UNDO** with old values | Crash after commit ⇒ **REDO** with new values | No full copy; needs both UNDO & REDO |

### Key Takeaways

- Shadow copy depends on a **single-sector, hardware-atomic** DB pointer write.
- Shadow copy is **simple but inefficient** and requires **serial** transactions.
- Log-based recovery uses **stable storage** (separate from DB storage) and follows the **write-ahead** rule (log before actual operation).
- **Deferred DB modification**: defer real writes until **after commit**; ignore logs if no commit record exists (only REDO needed).
- **Immediate DB modification**: apply real writes **while the transaction is active** (uncommitted modifications); log records carry **old + new** values → **UNDO** on crash before commit, **REDO** on crash after commit.
