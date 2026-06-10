# Lecture 20 — Concurrency Control & Locking

---

## 1. Why Concurrency Control?

When multiple transactions run **simultaneously**, problems arise if they access the same data without coordination:

- **Lost Update** — T1 and T2 both read A=100, both add 50, both write 150 → one update is lost (should be 200).
- **Dirty Read** — T1 reads data written by T2 that hasn't committed yet. If T2 rolls back, T1 read garbage.
- **Unrepeatable Read** — T1 reads A twice; T2 updates A in between → T1 gets different values for the same read.

**Solution:** Locks — force transactions to take turns on shared data.

---

## 2. Shared Lock (S-Lock) vs Exclusive Lock (X-Lock)

### Exclusive Lock
- Valid on a **single transaction** — locks either a **row or a page** depending on the data.
- Is a **read + write lock** (not just write — the transaction holding it can also read).
- **Only one** exclusive lock can exist on the same resource at a time.
- Any transaction that needs an X-lock must **wait** if another task currently owns an **X-lock OR an S-lock** on that resource.

### Shared Lock
- A **read-only lock** at the row level — ensures a record is not being updated during a read-only request.
- Also prevents updates between the time a record is read and the next sync point.
- **Many transactions** can hold an S-lock on the same resource simultaneously.
- A shared lock request must **wait** if:
  - Another task currently owns an **X-lock** on the resource.
  - Another task is **waiting for an X-lock** on a resource that already has an S-lock (to avoid X-lock starvation).
- An X-lock request must wait if **other tasks currently own S-locks** on the resource.

### Side-by-side comparison

| | Exclusive Lock (X) | Shared Lock (S) |
|---|---|---|
| **Also called** | Write lock | Read lock |
| **Used for** | Read + Write | Read only |
| **Granularity** | Row or page | Row level |
| **How many at once?** | Only one per resource | Many simultaneously |
| **Blocks others?** | Blocks all (S and X) | Blocks X-locks only |
| **When must it wait?** | If any S or X lock exists on the resource | If an X-lock exists, or an X-lock is pending |

### Lock Compatibility Matrix

|  | S | X |
|---|---|---|
| **S** | ✅ Compatible | ❌ Not compatible |
| **X** | ❌ Not compatible | ❌ Not compatible |

> Two reads can happen in parallel. Any write blocks everything else.

---

## 3. Two-Phase Locking (2PL)

A protocol that **guarantees serializability** (transactions produce the same result as if they ran one after another).

**Two phases:**

```
Phase 1 — Growing phase:
  Transaction can ACQUIRE locks (S or X).
  Cannot release any lock yet.

Phase 2 — Shrinking phase:
  Transaction can RELEASE locks.
  Cannot acquire any new lock.
```

```css
2PL Timeline:

  Lock count
      │         /\
      │        /  \
      │       /    \
      │──────/      \──────▶ time
             ↑      ↑
           lock    first
           point   release
           (peak)
```

- The **lock point** = moment the transaction holds its maximum locks = end of growing phase.
- Once you release even one lock → shrinking phase begins → no new locks allowed.

**Why it guarantees serializability:** transactions can be ordered by their lock points → equivalent serial schedule.

### Strict 2PL (most common in practice)
- All **X-locks held until commit/abort** (prevents dirty reads).
- Most real DBs (MySQL InnoDB, PostgreSQL) implement this variant.

---

## 4. Deadlock

Occurs when two or more transactions are each **waiting for the other to release a lock** → circular wait → neither can proceed.

```css
Deadlock example:

  T1 holds X-lock on A, wants X-lock on B
  T2 holds X-lock on B, wants X-lock on A

  T1 ──waiting──▶ B (held by T2)
  T2 ──waiting──▶ A (held by T1)
        ↑________________________|
              circular wait = deadlock
```

### Detection & Resolution
- DB periodically checks for cycles in the **wait-for graph**.
- If cycle found → pick a **victim** transaction → **abort it** → release its locks → other transaction proceeds.
- Victim is usually the one that has done the least work (cheapest to restart).

### Prevention (instead of detecting after the fact)
- **Wait-Die:** older transaction waits, younger one dies (aborts).
- **Wound-Wait:** older transaction wounds (forces abort) the younger one, younger waits.

> Most production DBs use **detection + abort** rather than prevention.

---

## 5. Quick Interview Summary

| Concept | One-liner |
|---|---|
| S-Lock | Multiple readers allowed simultaneously |
| X-Lock | One writer, blocks everyone else |
| 2PL | Acquire all locks before releasing any → guarantees serializability |
| Strict 2PL | Hold X-locks until commit → no dirty reads |
| Deadlock | Circular wait; resolved by aborting a victim transaction |
| Dirty Read | Reading uncommitted data from another transaction |
| Lost Update | Two transactions overwrite each other's changes |
