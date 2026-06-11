## 1. Why Concurrency Control?

When multiple transactions run **simultaneously** on the same data without coordination, three problems arise:

| Problem | What happens |
|---|---|
| **Lost Update** | T1 and T2 both read A=100, both add 50, both write 150 → should be 200, one update is lost |
| **Dirty Read** | T1 reads data written by T2 before T2 commits → if T2 rolls back, T1 read garbage |
| **Unrepeatable Read** | T1 reads A twice; T2 updates A in between → T1 gets different values for the same read |

**Lost Update** is a write-write conflict — two transactions both think their write is the final one.
**Dirty Read** is a read-write conflict across transaction boundaries — you're reading work-in-progress that may never actually commit.
**Unrepeatable Read** is also read-write, but within a single transaction — the same query returns different results because someone else changed the row mid-transaction.

**Solution:** Locks — force transactions to take turns on shared data.

---

## 2. Shared Lock (S-Lock) vs Exclusive Lock (X-Lock)

The core idea: readers shouldn't block each other, but a writer must have the resource to itself.

### Exclusive Lock (X-Lock)
- **Read + write lock** — the holder can both read and write.
- Only **one** X-lock can exist on a resource at a time.
- Must **wait** if any S-lock or X-lock already exists on the resource.
- Analogy: like taking a pen out of a shared jar to write — nobody else can touch it until you put it back.

### Shared Lock (S-Lock)
- **Read-only lock** — ensures a record isn't being updated while you're reading it.
- **Many transactions** can hold an S-lock on the same resource simultaneously — reads don't interfere with each other.
- Must **wait** if an X-lock exists on the resource, or if another transaction is already waiting for an X-lock (to prevent X-lock starvation).
- Starvation risk: if reads keep arriving continuously, a write request could wait forever. So once a write is queued, new reads also wait behind it.

### Comparison

| | Exclusive Lock (X) | Shared Lock (S) |
|---|---|---|
| **Also called** | Write lock | Read lock |
| **Used for** | Read + Write | Read only |
| **How many at once?** | Only one per resource | Many simultaneously |
| **Blocks others?** | Blocks all (S and X) | Blocks X-locks only |
| **When must it wait?** | If any S or X lock exists | If an X-lock exists or is pending |

### Lock Compatibility Matrix

| | S | X |
|---|---|---|
| **S** | ✅ Compatible | ❌ Not compatible |
| **X** | ❌ Not compatible | ❌ Not compatible |

> Two reads can happen in parallel. Any write blocks everything else.

---

## 3. Two-Phase Locking (2PL)

Having locks alone isn't enough — you also need a **discipline for when to acquire and release them**. Without a protocol, a transaction could release a lock early and let another transaction sneak in, breaking the illusion that transactions ran alone.

2PL is that discipline. It **guarantees serializability** — the interleaved execution produces the same result as some serial (one-after-another) order of those transactions.

**Phase 1 — Growing:** The transaction acquires locks (S or X). No locks are released yet.

**Phase 2 — Shrinking:** The transaction releases locks. No new locks can be acquired.

```
Lock count
    │         /\
    │        /  \
    │       /    \
    │──────/      \──────▶ time
           ↑      ↑
        lock     first
        point    release
        (peak)
```

- The **lock point** = the peak = end of the growing phase = maximum locks held.
- The moment you release even one lock → shrinking phase begins → no new locks allowed.
- Transactions can be ordered by their lock points → this ordering is equivalent to a serial schedule → **serializability guaranteed**.

Why does releasing a lock early break serializability? Because once you release a lock on X, another transaction can read or modify X. If your transaction later reads X again, you might see a different value — your transaction is no longer isolated, and the execution can't be mapped to any serial order.

### Strict 2PL (most common in practice)
Basic 2PL still allows dirty reads — a transaction could release an X-lock before it commits, letting another transaction read that uncommitted data. **Strict 2PL fixes this** by holding all X-locks until commit or abort. Only then are locks released. This is what MySQL InnoDB and PostgreSQL implement.

---

## 4. Deadlock

Occurs when two or more transactions are each **waiting for a lock held by the other** — neither can proceed, and they'll wait forever without intervention.

```
T1 holds X-lock on A, waiting for X-lock on B
T2 holds X-lock on B, waiting for X-lock on A

T1 ──waiting──▶ B (held by T2)
T2 ──waiting──▶ A (held by T1)
      ↑________________________|
            circular wait = deadlock
```

The reason this happens: transactions acquire locks one at a time as they need them, not all upfront. So T1 grabs A first, T2 grabs B first — then both try to grab the other's resource and freeze.

### Detection & Resolution
The DB maintains a **wait-for graph** where each node is a transaction and a directed edge T1 → T2 means "T1 is waiting for a lock held by T2." A **cycle** in this graph = deadlock. The DB periodically checks for cycles, picks a **victim** (usually the transaction that has done the least work, so it's cheapest to restart), **aborts it**, and releases its locks so the others can proceed.

### Prevention
Rather than letting deadlocks form and then resolving them, prevention schemes assign priorities (usually by timestamp — older = higher priority) and abort one transaction upfront when a conflict is detected:

- **Wait-Die:** if the requesting transaction is **older**, it waits; if it's **younger**, it dies (aborts and retries later). Older always gets priority.
- **Wound-Wait:** if the requesting transaction is **older**, it wounds (forces abort of) the younger holder; if it's **younger**, it waits. Again, older wins, but the older transaction acts aggressively rather than waiting.

The key difference: in Wait-Die the older transaction is passive (waits); in Wound-Wait the older transaction is aggressive (kicks out the younger). Both are deadlock-free because the priority is consistent — no circular dependency can form.

> Most production DBs use **detection + abort** rather than prevention, since prevention aborts transactions that might never have actually deadlocked.

---

## 5. Isolation Levels

Locks and 2PL guarantee full isolation (serializability), but **full isolation is expensive** — it means transactions often block each other waiting for locks. In practice, applications choose a **weaker isolation level** that allows some anomalies in exchange for higher concurrency and throughput.

SQL defines four standard isolation levels, each allowing a different set of anomalies.

### The Three Anomalies (recap)

| Anomaly | What happens |
|---|---|
| **Dirty Read** | Read uncommitted data from another transaction that may roll back |
| **Non-Repeatable Read** | Read the same row twice, get different values (another transaction updated it in between) |
| **Phantom Read** | Run the same range query twice, get different rows (another transaction inserted/deleted in between) |

### The Four Isolation Levels

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | How it works |
|---|---|---|---|---|
| **READ UNCOMMITTED** | ✅ Possible | ✅ Possible | ✅ Possible | No read locks at all — reads never block |
| **READ COMMITTED** | ❌ Prevented | ✅ Possible | ✅ Possible | S-locks acquired and **immediately released** after each read |
| **REPEATABLE READ** | ❌ Prevented | ❌ Prevented | ✅ Possible | S-locks held until end of transaction |
| **SERIALIZABLE** | ❌ Prevented | ❌ Prevented | ❌ Prevented | Full 2PL + range locks (or snapshot isolation) |

> **Default in most databases:** MySQL InnoDB uses **REPEATABLE READ**; PostgreSQL uses **READ COMMITTED**.

### Level-by-Level Explanation

#### READ UNCOMMITTED
- A transaction can read rows that another transaction has modified **but not yet committed**.
- If T2 rolls back after T1 read its dirty write → T1 made a decision based on data that never existed.
- Practically never used except in scenarios where absolute maximum throughput matters and stale reads are acceptable (e.g., approximate analytics dashboards).

```css
READ UNCOMMITTED — dirty read scenario:

  T1: BEGIN                             T2: BEGIN
                                        T2: UPDATE users SET bal=0 WHERE id=1
  T1: SELECT bal FROM users WHERE id=1
      → reads 0 (T2's uncommitted write!)
                                        T2: ROLLBACK  (bal is actually still 1000)
  T1: sees 0, acts on wrong data ❌
```

#### READ COMMITTED
- T1 can only read rows that T2 has **already committed**.
- Dirty reads are prevented. But if T1 reads the same row twice and T2 commits an update in between, T1 gets two different values → **non-repeatable read** still possible.
- S-locks are taken and dropped immediately after each read statement (not held for the transaction duration).

```css
READ COMMITTED — non-repeatable read still possible:

  T1: SELECT bal → 1000         (T2 hasn't touched it yet)
                                T2: UPDATE bal = 500; COMMIT
  T1: SELECT bal → 500          (same query, different result)
  T1: confused — within one transaction, balance changed ❌
```

#### REPEATABLE READ
- S-locks are held for the **entire duration** of the transaction. Once T1 reads a row, no other transaction can update that row until T1 commits.
- Non-repeatable reads are prevented. But a range query (`WHERE age > 25`) can still return different *sets* of rows if another transaction inserts a new row that matches — **phantom read** still possible.

```css
REPEATABLE READ — phantom read still possible:

  T1: SELECT * FROM orders WHERE amount > 100  → returns 5 rows
                                T2: INSERT INTO orders (amount=200); COMMIT
  T1: SELECT * FROM orders WHERE amount > 100  → returns 6 rows ❌
  (same query, same transaction, different row count)
```

#### SERIALIZABLE
- The highest level. Transactions execute as if they were **fully serial** — one at a time.
- Implemented via **range locks** (lock the entire range of the query, not just the rows that currently exist) or **snapshot isolation with conflict detection**.
- Prevents all three anomalies. Most expensive — maximum blocking.

```css
SERIALIZABLE — range lock prevents phantom:

  T1: SELECT * FROM orders WHERE amount > 100
      → DB places a range lock on "amount > 100"

  T2: INSERT INTO orders (amount=200)
      → T2 must wait — the range is locked by T1

  T1 commits → range lock released → T2 proceeds.
  T1 always sees a consistent snapshot. No phantoms.
```

### Choosing an Isolation Level

- **Most web applications** use **READ COMMITTED** — it prevents the worst anomaly (dirty reads) while allowing high concurrency. Non-repeatable reads are usually acceptable because most requests are short-lived.
- **Financial / inventory systems** that need to read-then-write based on a value (e.g., check balance, then debit) need at least **REPEATABLE READ** to avoid reading a stale balance.
- **SERIALIZABLE** is used when correctness is critical and the operation genuinely depends on no other transaction interfering at all (e.g., booking the last seat on a flight).

```css
ISOLATION LEVEL TRADE-OFF:

  READ UNCOMMITTED ─────────────────────────── SERIALIZABLE
       ↑                                              ↑
  Maximum throughput                        Maximum correctness
  Maximum concurrency                       Maximum blocking
  Minimum anomaly protection                Full anomaly protection
```

---

## 6. Quick Revision

| Concept | One-liner |
|---|---|
| S-Lock | Multiple readers allowed simultaneously |
| X-Lock | One writer, blocks everyone else |
| 2PL | Acquire all locks before releasing any → guarantees serializability |
| Strict 2PL | Hold X-locks until commit → no dirty reads |
| Deadlock | Circular wait; resolved by aborting a victim transaction |
| Dirty Read | Reading uncommitted data from another transaction |
| Non-Repeatable Read | Same row read twice gives different values within one transaction |
| Phantom Read | Same range query returns different row sets within one transaction |
| Lost Update | Two transactions overwrite each other's changes |
| READ UNCOMMITTED | No protection — allows dirty reads |
| READ COMMITTED | Prevents dirty reads; default in PostgreSQL |
| REPEATABLE READ | Prevents dirty + non-repeatable reads; default in MySQL |
| SERIALIZABLE | Prevents all anomalies; most expensive |
