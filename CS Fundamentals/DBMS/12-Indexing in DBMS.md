# 12 — Indexing in DBMS

> This file begins with the **tail end of the previous topic (Log-Based Recovery)** — completing **Immediate DB Modification** and introducing **Checkpoints** — before the main topic on **Indexing**.

---

## PART A — Log-Based Recovery (continuation)

### 1. Deferred DB Modification — Durability (completion)

- Suppose after `<T0, commit>` the system starts the deferred writes (writes 950 into A, 1050 into B) and then **the system fails**.
- On **restart**, read the logs in stable storage. The log contains `commit` → meaning T0 had committed.
- So we **re-execute** the corresponding operations and complete the commit → **durability is maintained.**
- **"If failure occurs while updating"** (during deferred update), we **read the logs again and re-perform** all those operations.
- This is **Deferred DB Modification**: write logs first → write `commit` → then perform actual update in DB.

> **Note on values:** A starts at 1000 and B starts at 1000; T0 transfers 50 so A → 950 and B → 1050.

```
DEFERRED MODIFICATION — DURABILITY SCENARIO:

  Stable Storage (survives crash):
  <T0,start> → <T0,A,950> → <T0,B,1050> → <T0,commit> ← commit is here!

  System crashes mid-write (after writing A but before writing B to actual DB):
  DB state: A = 950 ✅  B = 1000 ❌ (not yet written to actual DB)

  On Restart:
  → Read logs in stable storage
  → Find <T0, commit> → T0 had committed → must complete its writes
  → REDO: apply <T0,B,1050> → write B into actual DB
  → Durability guaranteed ✅

  KEY: Stable storage survives the crash. The commit record tells the
       system "this transaction was complete — finish its writes."
```

### 2. Immediate DB Modification

**Concept:** As the name suggests, modifications are made **immediately** — while the transaction is **still in the active state**, DB modifications are applied as the transaction runs.

- A write operation is **logged first**, and **immediately after logging**, the value is **written to the actual DB** (repeated updates happen on the real DB during execution).
- Modifications written by an active transaction are called **uncommitted modifications** (because the transaction may fail mid-way and need a **rollback**).

> **Difference from Deferred:** In deferred, DB is unchanged until commit. In immediate, DB is changed live during the transaction — which means if you abort, you must undo what you already wrote.

**Log format (extra field added):** Compared to deferred logs (which had only new value), immediate logs add an **old value** field:

```
<T0, A, 1000, 950>   # <Txn, item, OLD value, NEW value>
```

- **Why old value?** Because if you immediately update (wrote 950 over 1000) and the transaction then fails, atomicity requires a **rollback**, which needs the **old value** to restore. → `<old item value, new item value>`.

```
IMMEDIATE MODIFICATION — LOG FORMAT COMPARISON:

  Deferred:   <T0, A, 950>           ← only new value (no need to undo)
  Immediate:  <T0, A, 1000, 950>     ← old AND new value (need old to undo)

  Why? In immediate mode, the DB is already changed.
  If crash/abort happens, you need the OLD value to roll back.
```

**Failure handling:**

- **System failure / transaction abort BEFORE commit:**
  - Use the **old-value field** to **UNDO** the modification (restore A back to 1000, B back to 1000 — the values before T0 started).
  - → **Atomicity** maintained.
- **Transaction complete (commit written) then system crash:**
  - On restart, the log has `commit` → use the **new-value field** to **REDO** the transaction (replace old field with new field).
  - Only redo transactions that **have a `commit` log record**; if no `commit`, assume the transaction failed.
  - → **Durability** maintained.
- **Golden rule:** Update takes place **only after the log record is in stable storage** (write log first, then actual update — else recovery is impossible).

> **Why UNDO needs the old value:** In immediate mode, the DB is already modified mid-transaction. If the transaction aborts, the changes are already sitting in the real DB. The only way to reverse them is to know the original value — which is why the log stores `<Txn, item, OLD, NEW>` instead of just `<Txn, item, NEW>`.

```
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

### 3. Checkpoint (improved log-based recovery)

- Logs keep accumulating as thousands/lakhs of transactions run → **stable storage fills up** and **cost increases**. Real-time systems can't store unlimited old information.
- A **checkpoint** is basically an **announcement.** At a point where the **DB state is consistent** (all prior transactions are in **commit** state), create a checkpoint (e.g., `C1`, `C2`).
- If a later transaction fails, you **recover to that checkpoint position** — you don't need to scan all logs from the beginning.
- It builds on the base methods above.

```
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

> **Log-based recovery is better than shadow copy.** If the system fails between the `0x01` and `0x02` pointer write: the disk system performs **atomic operations on a particular block/sector**, so the DB pointer update is atomic.

---

## PART B — Indexing (Performance Optimization)

### 4. The Problem (Why Indexing?)

- A database's data is **stored on disk** inside **blocks** of a **data file** (a.k.a. main file / database file). E.g., the `Student` table lives in blocks on disk.
- You sit at the DBMS layer (laptop), write SQL queries (`SELECT … WHERE …`), and the DBMS **fetches** the data from the actual data file on disk and beautifies it on screen.
- **Disk access is slow.** Fetching tuples directly by scanning the data file is a slow process.

**Setup example (used throughout):**
- 10,000 records (tuples) → `Student` data file.
- Each block holds **10 records** → Total blocks = **10,000 / 10 = 1000 blocks**.
- Roll number = primary key → so the file is **sorted**.

**Searching without an index:**
- **Linear search** on the data file — slow.
- **Binary search** (since file is sorted on roll number) — better. But for 1 lakh / 1 crore records this still hits the disk a lot.

```
WITHOUT INDEX — searching for Roll No. 843:

  Data File on disk (1000 blocks, 10 records each):
  ┌──────┬──────┬──────┬──────┬─────────┬──────────┐
  │  b1  │  b2  │  b3  │  b4  │  ...    │  b1000   │
  │ 1-10 │11-20 │21-30 │31-40 │         │9991-10000│
  └──────┴──────┴──────┴──────┴─────────┴──────────┘
                                Binary search: ~10 disk reads
                                Linear search: up to 1000 disk reads

  Problem: Each disk read is slow. For millions of records → very slow.
```

### 5. The Index — Concept & Logic

An **index** is a **data structure** used to **locate data** quickly. Structure:

| Search Key | Base Pointer |
|---|---|
| 1 | b1 (block where roll 1 lives) |
| 11 | b2 |
| 21 | b3 |
| … | … |
| 9991 | b1000 |

- **Search Key** — the attribute on which we search (here, roll number). Can be a primary key, candidate key, or any attribute. It acts as a **pointer** to the block / data reference.
- **Base Pointer** — points to the **block** where the corresponding search key (and the records after it) live.
- The index file can live on **disk or RAM** (doesn't matter).

**How it speeds up search (example: find Roll 843):**
1. The **index file is always sorted** → apply **binary search** on the 1000 index entries.
2. Find that 843 falls under entry **841 → block b841**.
3. Go to that **single block** and apply **linear/binary search** inside it to retrieve roll 843's full tuple (name, age, course_id, etc.).

> **Effect:** Instead of binary search over 10,000 records, do binary search over **1000 index entries**, then search **one small block**. → Reduces **search effort and disk access**.

```
WITH INDEX — searching for Roll No. 843:

  Sparse Index (1 entry per block, 10 records per block):
  ┌────────────┬──────────────────────────────────────────┐
  │ Search Key │ Block Pointer (first roll of that block) │
  │     1      │ ──► b1   (contains rolls 1–10)           │
  │    11      │ ──► b2   (contains rolls 11–20)          │
  │   ...      │                                          │
  │   841      │ ──► b85  (contains rolls 841–850) ◄──┐  │
  │   851      │ ──► b86  (contains rolls 851–860)    │  │
  └────────────┴──────────────────────────────────────┼──┘
                                                       │
  Step 1: Binary search the 1000-entry index (~10 comparisons)
          843 lies between key 841 and 851 → b85 is the block ──┘
  Step 2: Read block b85 from disk (1 disk read)
  Step 3: Linear/binary search inside b85 (10 records) → find roll 843

  Total: ~11 operations vs potentially 1000 disk reads without index!
  The index narrows 10,000 records down to 10 records in one jump.
```

### 6. Properties of Indexing

- **Optimization** that reduces disk access. Helps **read/search operations** (`SELECT … WHERE …`) where data is retrieved from the data file. By default, most SQL implementations use indexing.
- **Search Key:** can be primary key, candidate key, or any attribute. It's a pointer addressing the block / data reference for **direct access**.
- **Optional:** For small databases, indexing isn't needed; it pays off only when the database is **large**. But it **increases access speed.**
- **Secondary means, not primary means** of access: the tuple can still be accessed directly (linear/binary search); the index is an **optional, more optimized** path.
- **Index file is always sorted** — so binary search can be applied. An unsorted index gives no benefit (forces linear search). **Binary search is always better than linear search.**
- The index file must be **small** to be beneficial — if you index every record identically to the data file, there's no gain.

---

### 7. Dense vs Sparse Index

- **Dense Index:** Contains an index record for **every search key** value in the data file. (All of 1, 2, 3, … 10000 have entries.)
- **Sparse Index:** An index record appears for **only some** of the search keys. E.g., store only the **first search key of each block** (1, 11, 21, … — entry per block, ~101 gap).

Which is better → **depends case to case** (covered below).

```
DENSE vs SPARSE INDEX (10,000 records, 10 per block):

  Dense Index:                      Sparse Index:
  ┌────┬────────┐                   ┌────┬────────┐
  │  1 │ ──► b1 │                   │  1 │ ──► b1 │
  │  2 │ ──► b1 │                   │ 11 │ ──► b2 │
  │  3 │ ──► b1 │                   │ 21 │ ──► b3 │
  │ .. │  ...   │                   │ .. │  ...   │
  │9999│ ──►b1000│                  │9991│ ──►b1000│
  └────┴────────┘                   └────┴────────┘
  10,000 entries (1 per record)     1,000 entries (1 per block)
  More space, faster pinpoint       Less space, slightly more
                                    work inside block
```

---

### 8. Types of Indexing

Broadly, **two types**: **Primary Indexing** and **Secondary Indexing**.

#### 8.1 Primary Indexing (a.k.a. Clustering Index)

**Definition / Criterion:** Applied when the **data file is in sequential (sorted) order** based on **some search key** (one attribute). The **only condition** for primary indexing is: **the data file must be sorted on some one search key.**

- **Important caveat:** "Primary index" is **sometimes** loosely used to mean "an index on the **primary key**." This usage is **non-standard and should be avoided.** The search key for a primary index can be **primary key OR non-primary key** — there's no restriction that it must be the primary key.
- Why a file is sorted on only one key: a `Student(roll_no, name, age)` table sorted on roll_no will be **unsorted** on age (and vice versa). You can keep it sorted on **only one** attribute.

Primary indexing has **two cases** (based on the sort attribute):

**Case 1 — Sorted on a KEY attribute (unique values) → Sparse Index**
- Since the file is sorted and the key is unique, you only need **one entry per block** (store the block's first/base pointer).
- To find roll 14: locate block via roll 11's entry, then 12, 13, 14… are all in that same block (search within).
- → No need to store every entry → **Sparse Indexing.**
- **Number of index entries = Number of blocks** (e.g., 1000 entries for 10,000 records).

```
PRIMARY INDEX — CASE 1 (Sorted on KEY = roll_no, unique):

  Data File (sorted by roll_no):        Index File (Sparse):
  ┌──────────────────────────┐          ┌──────┬────────┐
  │ b1: roll 1,2,3,...,10    │ ◄───────│  1   │ ──► b1 │
  │ b2: roll 11,12,...,20    │ ◄───────│ 11   │ ──► b2 │
  │ b3: roll 21,22,...,30    │ ◄───────│ 21   │ ──► b3 │
  │ ...                      │          │ ...  │  ...   │
  │b1000: roll 9991,...,10000│ ◄───────│9991  │ ──►b1000│
  └──────────────────────────┘          └──────┴────────┘
                                        1000 entries (not 10,000)
  Find roll 14 → index entry "11" → b2 → search b2 for roll 14 ✅
```

**Case 2 — Sorted on a NON-KEY attribute (duplicate values possible) → Dense Index**
- Data file sorted on a non-key attribute → values can **repeat / cluster** (e.g., `1 1 1 2 3 3 3 4 4 4 5`).
- Store an entry for **each unique value**: the base pointer to the block where that value **first appears**. Since sorted, all matching duplicates follow contiguously (move to next addresses until the value changes).
- **Use case:** `GROUP BY` queries — e.g., department-wise average salary, all employees in a department, course-wise enrolled students. You need all records for a non-key value grouped together.
- → Because every **unique value** is stored → **Dense Indexing.**
- **Number of index entries = number of UNIQUE non-key attribute values** in the data file.

```
PRIMARY INDEX — CASE 2 (Sorted on NON-KEY = dept_id, duplicates exist):

  Data File (sorted by dept_id):         Index File (Dense on unique values):
  ┌────────────────────────────┐         ┌────────┬────────┐
  │ b1: dept=1, dept=1, dept=1 │ ◄──────│   1    │ ──► b1 │  ← first dept=1
  │ b2: dept=1, dept=2, dept=2 │         │   2    │ ──► b2 │  ← first dept=2
  │ b3: dept=2, dept=3, dept=3 │         │   3    │ ──► b3 │  ← first dept=3
  └────────────────────────────┘         └────────┴────────┘

  To find all dept=2 records:
  Index → b2 (first occurrence) → read b2, b3 until dept changes to 3.
  Perfect for GROUP BY dept queries!
```

> **Note:** Some writers/blogs call Case 2 a separate "Clustering Index" (not part of primary indexing). These notes treat **Clustering Index = Primary Indexing** — because primary indexing's only criterion is "sorted on one search key."

**Summary of primary indexing:**
- Sorted on **key** → **Sparse** (entries = #blocks).
- Sorted on **non-key** → **Dense** (entries = #unique values).

---

### 9. Multi-Level Indexing

- Still an example of **primary indexing** (file sorted on a search key).
- When the file is **very large** (e.g., population of India), the **first-level index itself becomes huge.** So you apply **further indexing on top of the index** → **Multi-Level Indexing** (index on an index).

**Example (Two-level sparse index):**
- **Data file:** roll numbers 100, 101, 102, … on disk.
- **Inner index (1st level):** stores 100 → data block (100–109), 110 → next data block (110–119), … (analogous to the earlier 1, 11, 21 example).
- **Outer index (2nd level):** groups the inner index — 100 → (entries 100 up to before 200), 200 → (200 to before 300), 300 → …

**Lookup (find roll 209):**
1. Go to **outer index** → 200's entry.
2. The outer pointer leads to the **inner index block** covering 200–299.
3. Find 209's pointer → go to the **actual data block** and retrieve 209.

**Why store like this:** The first-level index table got **too big** to keep entirely in RAM, so it's stored on **hard disk**, and a **smaller (outer) index** is kept in **RAM**. The data file was always on hard disk.

```
MULTI-LEVEL INDEXING — find roll 209:

  The inner index works like the basic sparse index (1 entry per data block,
  each data block holds 10 rolls). So inner block B (covering 200–299) has:
  200 → data block for rolls 200–209
  210 → data block for rolls 210–219  ... and so on.

  RAM (small enough to keep here):
  ┌───────────────────────────────────────┐
  │  Outer Index (2nd level)              │
  │  100 ──► inner index block A          │
  │  200 ──► inner index block B ──────┐  │
  │  300 ──► inner index block C       │  │
  └────────────────────────────────────┼──┘
                                        │
  DISK:                                 ▼
  ┌──────────────────────────────────────────┐
  │  Inner Index block B (1st level):        │
  │  200 ──► data block D1 (rolls 200–209)──┐│ ← 209 falls here (between 200 and 210)
  │  210 ──► data block D2 (rolls 210–219)  ││
  │  220 ──► data block D3 (rolls 220–229)  ││
  └─────────────────────────────────────────┼┘
                                             │
  ┌──────────────────────────────────────────▼┐
  │  Data Block D1 (disk):                    │
  │  roll 200, 201, 202, ..., 209, ...        │ ← find roll 209 here ✅
  └───────────────────────────────────────────┘

  Step 1: Binary search outer index (in RAM) → 209 is in range 200–299 → block B
  Step 2: Read inner index block B from disk → 209 between 200 and 210 → D1
  Step 3: Read data block D1 from disk → find roll 209

  Only 2 disk reads (inner index + data block).
  Outer index stays in RAM — very fast lookup regardless of data size.
```

---

### 10. Secondary Indexing (a.k.a. Non-Clustering Index)

**Definition:** Applied on data files that are **unsorted** (with respect to the indexed attribute). Until now, everything (primary, multi-level) was on sorted files where binary search worked. Here you **cannot binary-search the data file.**

- **When needed:** A data file is typically already sorted on **one** attribute (e.g., roll number → primary indexing). For **any other attribute** (age, name, address…), the file is **unsorted**. To index on such a **secondary key**, use **secondary indexing.**
- The search key here can be a **key or non-key** — doesn't matter, because the file is unsorted anyway.

**Mechanics (file unsorted):**
- Since unsorted, after finding value 1 you can't assume 2, 3… follow. The value could be scattered across blocks.
- So you must store the **base pointer for EVERY search key** (1, 2, 3, … 100…). → This is **always a Dense Index.**
- **Number of entries = number of records in the data file** (= number of unique search key values).

> **Clarification on "number of entries":** When the search key is unique (e.g., age is unique for every record), entries = total records = unique values — these are the same. When the search key has duplicates (e.g., multiple students share the same age), entries = number of **unique** values, and each entry holds a linked list of all block pointers for that value. The duplicate case is explained separately below.

**Case: search key has duplicate (repeating) values (unsorted):**
- A value (e.g., 1) may appear in blocks b1, b7, b19…
- Store **one entry per unique value**, and that entry points to a **linked list** (or a further index table) of all block pointers where the value occurs.
- → Index table entries = **unique search key values**; each entry has a **linked list** of locations.

**Benefit of secondary indexing (interview question):**
- Without an index on an unsorted file, you'd do a **linear search** over (e.g.) 1 lakh records.
- With a secondary index: the **index table itself is sorted**, so you can **binary search** the index (e.g., to find entry 89), then jump to the corresponding block pointer. → Big speedup despite the data file being unsorted.

```
SECONDARY INDEX — unsorted data file, indexing on "age":

  Data File (unsorted by age — sorted by roll_no):
  ┌────────────────────────────────────────────┐
  │ b1:  roll=1,age=20  | roll=2,age=17  | ... │
  │ b7:  roll=71,age=20 | roll=72,age=22 | ... │
  │ b19: roll=191,age=20| roll=192,age=18| ... │
  └────────────────────────────────────────────┘
  age=20 appears in b1, b7, b19... (scattered!)

  Secondary Index (SORTED, dense):
  ┌──────┬──────────────────────────────────────┐
  │ age  │ Pointer(s)                           │
  │  17  │ ──► b1                               │
  │  18  │ ──► b19                              │
  │  20  │ ──► [b1 → b7 → b19 → NULL] (linked) │
  │  22  │ ──► b7                               │
  └──────┴──────────────────────────────────────┘
           ↑ Index is SORTED → binary search possible!

  Find all age=20 records:
  Step 1: Binary search sorted index → find age=20 entry
  Step 2: Follow linked list → read b1, b7, b19
  Much faster than linear scan of all 10,000 records!
```

---

### 11. Advantages & Limitations of Indexing

**Advantages:**
- **Faster access and retrieval** of data.
- **I/O is less** (fewer disk accesses).

**Limitations:**
- **Additional space** required to store the index table.
- Indexing **decreases performance in INSERT, DELETE, and UPDATE** queries (the index must also be maintained/updated on every write).

---

## Summary Tables

### Recovery (Part A)

| Method | Idea | Atomicity | Durability |
|---|---|---|---|
| **Deferred Modification** | Log first, apply DB writes only after `commit` | No commit ⇒ ignore logs | Redo from log after commit |
| **Immediate Modification** | Log (`old,new`) then update DB live | UNDO using old value | REDO using new value (if `commit` exists) |
| **Checkpoint** | Periodic consistent-state markers | — | Recover from nearest checkpoint, not whole log |

### Indexing (Part B)

| Type | Data file | Index density | # Entries | Search inside data |
|---|---|---|---|---|
| **Primary (Sparse)** | Sorted on **key** | Sparse | # blocks | Binary search |
| **Primary (Dense / Clustering)** | Sorted on **non-key** | Dense | # unique values | Binary search |
| **Multi-Level** | Sorted (large) | Index on index | reduced top level | Binary search at each level |
| **Secondary (Non-Clustering)** | **Unsorted** | Always Dense | # records / unique values (+ linked list for dupes) | Binary search on **sorted index**, then jump |

### Key Takeaways
- Index file is **always sorted** → enables **binary search**.
- **Primary index** requires the data file **sorted on a search key** (key→sparse, non-key→dense/clustering).
- **Secondary index** is for **unsorted** files and is **always dense**; benefit = binary-searchable sorted index over an unsorted data file.
- **Multi-level** indexing = index on an index, used when the index itself grows too large for RAM.
