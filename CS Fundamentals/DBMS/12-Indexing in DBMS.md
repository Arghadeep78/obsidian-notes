## Indexing (Performance Optimization)

### 4. The Problem (Why Indexing?)

- A database's data is **stored on disk** inside **blocks** of a **data file** (aka main file / database file). E.g., the `Student` table lives in blocks on disk.
- You sit at the DBMS layer (laptop), write SQL queries (`SELECT … WHERE …`), and the DBMS **fetches** the data from the actual data file on disk and beautifies it on screen.
- **Disk access is slow.** Fetching tuples directly by scanning the data file from disk is a *slow* process.

**Setup example (used throughout):**
- 10,000 records (tuples) → `Student` data file.
- Each block holds **10 records** → Total blocks = **10,000 / 10 = 1000 blocks**.
- Roll number = primary key → so the file is **sorted**.

**Searching without an index:**
- **Linear search** on the data file — slow.
- **Binary search** (since file is sorted on roll number) — better. But for 1 lakh / 1 crore records this still hits the disk a lot.

```css
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

| Search Key | Base Pointer                     |
| ---------- | -------------------------------- |
| 1          | b1 (block where roll 1-10 lives) |
| 11         | b2                               |
| 21         | b3                               |
| …          | …                                |
| 9991       | b1000                            |

- *A **block** (also called a **page**) is the fixed-size unit of data transfer between disk and main memory in a DBMS. The two terms mean the same thing — "block" is common in older/academic DBMS literature; "page" is common in modern DB engines. These notes use both interchangeably.*

- **Search Key** — the attribute on which we search (here, roll number). Can be a primary key, candidate key, or any attribute.
- **Base Pointer** — points to the **block on disk** where the corresponding records live. It is a physical address (block number + offset) that lets the DBMS seek directly to that location on disk.
- The index file is **stored on disk**, just like the data file. It gets loaded into RAM (buffer pool) when accessed — the same way data blocks are.

**How it speeds up search (example: find Roll 843):**
1. The **index file is always sorted** → apply **binary search** on the 1000 index entries.
2. Find that 843 falls under entry **841 → block b85** (the block covering rolls 841–850).
3. Go to that **single block** and apply **linear/binary search** inside it to retrieve roll 843's full tuple (name, age, course_id, etc.).

> **Effect:** Instead of binary search over 10,000 records, do binary search over **1000 index entries**, then search **one small block**. → Reduces *search effort and disk access.*

```css
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
---

### Heap File — How Data Is Stored Without Ordering

Before any sorting or indexing is applied, a table's records are stored in a **heap file** — the default storage structure in most DBs.

- Records are written **in insertion order** — no sorting, no arrangement.
- New records go to the **first block with free space**, or a new block is allocated at the end.
- There is **no inherent order** between records. Record 1000 may live in block 3; record 2 may live in block 50.

```css
HEAP FILE — 8 records, 4 per block:

  Block b1: [ roll=5 | roll=2 | roll=8 | roll=1 ]   ← insertion order, not sorted
  Block b2: [ roll=6 | roll=3 | roll=7 | roll=4 ]
  Block b3: [ roll=9 | (empty) | (empty) | (empty) ]  ← new records go here next
```

- **Search on a heap file** = full linear scan (every block must be checked) → slow.
- **This is exactly why indexing exists** — an index built on top of a heap file gives you a sorted, searchable structure without having to physically sort or reorganise the heap.

---

### How a Block Pointer Actually Works (Index → Disk Navigation)

When the index says "Search Key 841 → b85", what does that pointer actually mean and how does the DBMS use it?

- A **block pointer** is a physical address: typically a **block number** (e.g., block 85) within the data file.
- The DBMS knows the **block size** (e.g., 4 KB). So block 85 starts at byte offset `85 × 4096` in the file.
- The DBMS issues a **disk seek** directly to that offset — it does not scan blocks 1 through 84. This is the direct access benefit of indexing.
- The entire block is read into the **buffer pool (RAM)** in one I/O operation.
- The CPU then searches within that block (10 records) entirely in RAM — no further disk I/O.

```css
INDEX POINTER → DISK NAVIGATION:

  Index entry:  841 ──► block 85
                              │
                              ▼
  Data file on disk:
  ┌────┬────┬─────┬─────┬──────────────────┬──────┐
  │ b1 │ b2 │ ... │ b84 │       b85        │ b86  │
  │    │    │     │     │ rolls 841–850     │      │
  └────┴────┴─────┴─────┴──────────────────┴──────┘
                              ▲
                        DBMS seeks directly here
                        (offset = 85 × block_size)
                        reads b85 into RAM buffer pool
                        searches 10 records in RAM → finds roll 843
```

**Without the index:** DBMS would read b1, b2, b3… sequentially until it found roll 843 (up to 1000 disk reads).
**With the index:** 1 lookup in the index (in RAM or 1 disk read) + 1 disk read for b85 = **2 operations**.

---

### Simplified view

1. The DBMS uses the index to determine which **block** contains the required record.
2. It checks whether that block is already in the **buffer pool (RAM)**.
3. If not, it reads the entire block from **disk** into the buffer pool.
4. The CPU accesses the required record from that block in RAM.

- Databases store data in **blocks** (fixed-size units, also called pages).
- The DBMS performs disk I/O at the **block level**, not the individual row level.
- A block may contain many rows.
- When a row is needed, the DBMS loads the **entire block** containing that row into the buffer pool.

**What lives where:**
- **Disk (permanent):** data file (all table blocks) + index file (all index blocks) — both stored on disk always.
- **RAM / buffer pool (temporary):** blocks from the data file and index file get copied here on demand when accessed, and evicted when RAM fills up.
- **CPU cannot read from disk directly** — data must be in RAM first. The flow is always: disk → RAM (buffer pool) → CPU. Even if you know the exact `(block, slot)` of a record, the block must be loaded into RAM before the CPU can touch it. The index minimises how many blocks need to be loaded, but cannot skip this step.
- **Index in RAM over time:** because the index file is much smaller than the data file, the buffer pool tends to hold most of it after the DB has been running a while → index reads become cache hits (no disk I/O).
- **Multi-level outer index:** small enough to be kept pinned in RAM permanently — only the inner index blocks and data blocks need disk reads.

---
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
- **Sparse Index:** An index record appears for **only some** of the search keys. E.g., store only the **first search key of each block** (1, 11, 21, … — one entry per block, gap of 10).

Which is better → **depends case to case** (covered below).

```css
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

##### Case 1 — Sorted on a KEY attribute (unique values) → Sparse Index
- Since the file is sorted and the key is unique, you only need **one entry per block** (store the block's first/base pointer).
- To find roll 14: locate block via roll 11's entry, then 12, 13, 14… are all in that same block (search within).
- → No need to store every entry → *Sparse Indexing.*
- **Number of index entries = Number of blocks** (e.g., 1000 entries for 10,000 records).

```css
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

##### Case 2 — Sorted on a NON-KEY attribute (duplicate values possible) → Dense Index
- Data file sorted on a non-key attribute → values can **repeat / cluster** (e.g., `1 1 1 2 3 3 3 4 4 4 5`).
- Store an entry for **each unique value**: the base pointer to the block where that value **first appears**. Since sorted, all matching duplicates follow contiguously (move to next addresses until the value changes).
- **Use case:** `GROUP BY` queries — e.g., department-wise average salary, all employees in a department, course-wise enrolled students. You need all records for a non-key value grouped together.
- → Because every **unique value** is stored → **Dense Indexing.**
- **Number of index entries = number of UNIQUE non-key attribute values** in the data file.

```css
## Clustering Index (Sorted on Non-Key Attribute)
### Data File (sorted by Dept_ID)
┌─────────────────────────────┐
│ B1 │ 1 │ 1 │ 2 │ 3 │
├─────────────────────────────┤
│ B2 │ 3 │ 3 │ 4 │ 4 │ 4 │
├─────────────────────────────┤
│ B3 │ 5 │ ... │ 8 │ ... │
└─────────────────────────────┘
### Clustering Index
┌─────────┬─────────────┐
│ Dept_ID │ Block Ptr   │
├─────────┼─────────────┤
│    1    │ → B1        │
│    2    │ → B1        │
│    3    │ → B1        │
│    4    │ → B2        │
│    5    │ → B3        │
│   ...   │   ...       │
└─────────┴─────────────┘

### Example Search: Dept_ID = 3

Index:
3 → B1

Read blocks sequentially:

B1 : 1 1 2 3
           ↑ first 3

B2 : 3 3 4 4 4
      ↑ continue reading

Stop when value changes from 3 to 4.
```

> **Note:** Some writers/blogs call Case 2 a separate "Clustering Index" (not part of primary indexing). These notes treat **Clustering Index = Primary Indexing** — because primary indexing's only criterion is "sorted on one search key."

**Summary of primary indexing:**
- Sorted on **key** → **Sparse** (entries = #blocks).
- Sorted on **non-key** → **Dense** (entries = #unique values).

---

#### 8.2. Multi-Level Indexing

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
![[Pasted image 20260610102430.png]]

```css
MULTI-LEVEL INDEX — find roll 209:

  Outer index (in RAM):          Inner index (on disk):      Data file (on disk):
  ┌─────┬──────────┐             ┌─────┬──────────┐          ┌──────────────────┐
  │ 100 │ ──► IB-A │             │ 200 │ ──► D-200│          │ D-200: 200–209   │
  │ 200 │ ──► IB-B │ ──────────► │ 210 │ ──► D-210│ ──────► │ D-210: 210–219   │
  │ 300 │ ──► IB-C │             │ ... │   ...    │          │ ...              │
  └─────┴──────────┘             └─────┴──────────┘          └──────────────────┘

  Step 1: Binary search outer index (in RAM) → 209 is in range 200–299 → IB-B
  Step 2: Read inner index block IB-B from disk → 209 between 200 and 210 → D-200
  Step 3: Read data block D-200 from disk → find roll 209

  Only 2 disk reads (1 inner index block + 1 data block).
  Outer index stays in RAM — fast first lookup regardless of data size.
```

---

### 10. Secondary Indexing (a.k.a. Non-Clustering Index)

**Definition:** Applied on data files that are **unsorted** (with respect to the indexed attribute). Until now, everything (primary, multi-level) was on sorted files where binary search worked. Here you **cannot binary-search the data file.**

- **When needed:** A data file is typically already sorted on **one** attribute (e.g., roll number → primary indexing). For **any other attribute** (age, name, address…), the file is **unsorted**. To index on such a **secondary key**, use **secondary indexing.**
- The search key here can be a **key or non-key** — doesn't matter, because the file is unsorted anyway.

**Mechanics (file unsorted):**
- Since unsorted, after finding value 1 you can't assume 2, 3… follow. The value could be scattered across blocks.
- So you must store the **base pointer for EVERY search key** (1, 2, 3, … 100…). → This is **always a Dense Index.**
- **Number of entries = number of records in the data file** (only true when the search key is unique — every record has a distinct value, so entries = records = unique values).

> **Clarification on "number of entries":** When the search key is unique (e.g., age is unique for every record), entries = total records = unique values — these are the same. When the search key has duplicates (e.g., multiple students share the same age), entries = number of **unique** values, and each entry holds a linked list of all block pointers for that value. The duplicate case is explained separately below.

**Case: search key has duplicate (repeating) values (unsorted):**
- A value (e.g., 1) may appear in blocks b1, b7, b19…
- Store **one entry per unique value**, and that entry points to a **linked list** (or a further index table) of all block pointers where the value occurs.
- → Index table entries = **unique search key values**; each entry has a **linked list** of locations.

**Benefit of secondary indexing (interview question):**
- Without an index on an unsorted file, you'd do a **linear search** over (e.g.) 1 lakh records.
- With a secondary index: the **index table itself is sorted**, so you can *binary search* the index (e.g., to find entry 89), then jump to the corresponding block pointer to do *linear search* on it. → Big speedup despite the data file being unsorted.

```css
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

*Index files are usually stored on disk, but because they are much smaller than the data file, searching the index first greatly reduces the number of disk block accesses needed to find a record.*

---

## Summary Tables

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
- **Multi-level** indexing = index on an index, used when the index itself grows too large, Outer index often on RAM as it is tiny and can be easily cashed.

---

## B-Tree and B-Tree Indexing

### Why B-Tree?

All the indexing types above describe *what* to index and *how dense* the index is — but they don't describe the **data structure** used to store and search the index entries themselves. In real databases, the index is stored as a **B-Tree** (Balanced Tree).

The binary search on a sorted index file (as described above) assumes the entire index fits in RAM or that you can jump directly to the middle. On disk, that's not how it works — you can only read one block at a time. B-Tree is designed specifically for **disk-based search**: it keeps the tree **short and wide** so only a few disk reads are needed to reach any record.

### B-Tree Structure

A B-Tree of **order `m`** (also called degree `m`) means each node can hold at most `m-1` keys and have at most `m` children.

```css
B-TREE OF ORDER 5 (each node: max 4 keys, max 5 children):

                    [  30  |  70  ]
                   /        |       \
         [ 10 | 20 ]   [ 40 | 50 | 60 ]   [ 80 | 90 ]
        /   |   |  \       ...              ...
      [.] [.] [.] [.]
   (leaf nodes — actual data pointers)
```

- Every node (except root) has **at least ⌈m/2⌉ children** — this prevents the tree from becoming too sparse and keeps it balanced.
- All **leaf nodes are at the same depth** — the tree is always balanced → search time is always O(log n).
- Each key in an internal node acts as a **separator**: keys to the left are smaller, keys to the right are larger.
- Each node maps to **one disk block** — reading one node = one disk I/O.

### How B-Tree Search Works

To find a record with key = 45 in the tree above:
1. Read root block → `[30 | 70]` → 45 is between 30 and 70 → go to middle child.
2. Read middle child → `[40 | 50 | 60]` → 45 is between 40 and 50 → go to leaf.
3. Read leaf → find 45's record pointer → **one more disk read** to get the actual data.

Total: **3–4 disk reads** regardless of table size. For a B-Tree of order 100 (typical), a tree of height 3 covers 100³ = **1 million keys**. Height 4 covers 100 million. The tree stays extremely flat.

```css
HEIGHT vs RECORDS (order 100):

  Height 1: root only               → up to 99 keys
  Height 2: root + children         → up to ~10,000 keys
  Height 3: root + 2 levels         → up to ~1,000,000 keys
  Height 4: root + 3 levels         → up to ~100,000,000 keys

  Finding any record in a 100M-row table = 4 disk reads.
```

### B-Tree vs B+ Tree

Most real databases (MySQL InnoDB, PostgreSQL) actually use a **B+ Tree**, a variant:

| | B-Tree | B+ Tree |
|---|---|---|
| **Data pointers** | In every node (internal + leaf) | **Only in leaf nodes** |
| **Internal nodes** | Store keys + data pointers | Store keys only (more keys per node → shorter tree) |
| **Leaf nodes** | No links between them | **Linked list** across all leaves |
| **Range queries** | Expensive (tree traversal) | Fast — follow the leaf linked list |

```css
B+ TREE LEAF LINKED LIST (range query: keys 20 to 50):

  Leaves: [10|15|20] → [25|30|35] → [40|45|50] → [55|60] → ...
                          ↑ start here              ↑ stop here
  
  Just scan the linked list — no need to traverse the tree again.
  Perfect for: WHERE age BETWEEN 20 AND 50, ORDER BY, range scans.
```

> **Rule of thumb:** When someone says "B-Tree index" in a DB context (MySQL, Postgres, etc.), they almost always mean **B+ Tree**.

### B-Tree Index Formation

When you run `CREATE INDEX idx_name ON table(column)`:
1. The DB **sorts** all values of that column.
2. It builds the B+ Tree bottom-up: fills leaf nodes with (key, row_pointer) pairs, then builds internal nodes as separators.
3. The tree is written to disk as a separate index file (same block structure as the data file).
4. On `INSERT` / `UPDATE` / `DELETE`, the DB **updates the tree** — splits nodes if a leaf overflows, merges if it underflows. This is why writes are slower with indexes.

### When to Use an Index (and When Not To)

**Use an index when:**
- The column appears frequently in `WHERE`, `JOIN ON`, `ORDER BY`, or `GROUP BY`.
- The table is large (thousands+ of rows) and queries are selective (returning a small fraction of rows).
- The column has high **cardinality** (many distinct values — e.g., user_id, email). Low-cardinality columns (e.g., gender with 2 values) make the index nearly useless — the DB would still scan half the table.

**Avoid an index when:**
- The table is small — a full table scan is faster than the overhead of index lookup.
- The column is updated very frequently — every write must also update the B-Tree.
- The query returns a large fraction of rows — the DB may ignore the index anyway and do a full scan (cheaper than random I/O for each indexed row).
- Heavy write workload (bulk inserts, ETL pipelines) — indexes slow down every write.

```css
INDEX SELECTIVITY — why cardinality matters:

  Table: 1,000,000 users

  Index on email (cardinality ~1,000,000):
    WHERE email = 'x@y.com' → index finds 1 row → 2 disk reads. GREAT.

  Index on gender (cardinality = 2):
    WHERE gender = 'M' → index finds 500,000 rows → 500,000 random disk reads.
    Full table scan would read ~100,000 blocks sequentially → FASTER.
    DB optimizer will skip the index.
```

---

## Composite Index

### What Is a Composite Index?

An index on **more than one column**. Instead of one search key, the index is built on an ordered tuple of columns.

```sql
CREATE INDEX idx_name_age ON users(last_name, first_name, age);
```

The B+ Tree is built by sorting rows **first by last_name, then by first_name within the same last_name, then by age within the same last_name + first_name**. The sort order is lexicographic — exactly like how a phone book is sorted.

```css
COMPOSITE INDEX ON (last_name, first_name, age):

  Index entries (sorted lexicographically):
  ┌───────────┬────────────┬─────┬────────────┐
  │ last_name │ first_name │ age │ row_ptr    │
  ├───────────┼────────────┼─────┼────────────┤
  │ Sharma    │ Anil       │  22 │ ──► row 45 │
  │ Sharma    │ Anil       │  28 │ ──► row 12 │
  │ Sharma    │ Raj        │  25 │ ──► row 87 │
  │ Singh     │ Arjun      │  30 │ ──► row 3  │
  │ Singh     │ Priya      │  19 │ ──► row 61 │
  └───────────┴────────────┴─────┴────────────┘
```

### The Left-Prefix Rule

This is the most important rule for composite indexes. **The index can only be used if the query filters on a prefix of the indexed columns, starting from the leftmost column.**

```css
Index: (last_name, first_name, age)

  ✅  WHERE last_name = 'Sharma'
        → leftmost column used → index scan works

  ✅  WHERE last_name = 'Sharma' AND first_name = 'Anil'
        → first two columns used → index scan works

  ✅  WHERE last_name = 'Sharma' AND first_name = 'Anil' AND age = 22
        → all three columns used → index scan works (most selective)

  ❌  WHERE first_name = 'Anil'
        → skips last_name → index cannot be used, full table scan

  ❌  WHERE age = 22
        → skips both → index cannot be used

  ⚠️  WHERE last_name = 'Sharma' AND age = 22
        → last_name is used, but age skips first_name → index used only
          for last_name part; age filter applied in-memory after that
```

Why? Because the index is sorted by last_name first. If you don't filter on last_name, the relevant rows are **scattered across the entire index** — there's no contiguous range to scan.

### Column Order Matters

Put the **most selective column first** only if it's used in most queries. More practically: put the column that appears in the most `WHERE` clauses first, then the next most common, etc.

```css
Choosing column order for index (users table, common queries):

  Query A: WHERE last_name = ?               (very common)
  Query B: WHERE last_name = ? AND age = ?   (common)
  Query C: WHERE age = ?                     (rare)

  Best index: (last_name, age)
    → Query A uses left prefix ✅
    → Query B uses both columns ✅
    → Query C cannot use index ❌ (but it's rare, acceptable)

  If you had indexed (age, last_name):
    → Query A cannot use index ❌
    → Query B uses both columns ✅
    → Query C uses left prefix ✅
    → Worse overall because Query A is the most common.
```

### Composite Index vs Multiple Single-Column Indexes

A composite index `(a, b)` is **not** the same as two separate indexes on `a` and `b`.

- `(a, b)` — one index, sorted by a then b. Directly handles `WHERE a = ? AND b = ?` in one B-Tree lookup.
- Separate indexes on `a` and `b` — the DB has to look up both separately, collect two sets of row pointers, and **intersect** them. Slower.

> For queries that always filter on both columns together, a composite index is better. For queries that filter on either column independently, separate indexes may be better.

### When to Use a Composite Index

- Multiple columns are consistently used together in `WHERE` clauses.
- You want a **covering index** — the index contains all columns the query needs, so the DB never touches the actual table (the index itself answers the query).

```css
COVERING INDEX example:

  Query:  SELECT first_name, age FROM users WHERE last_name = 'Sharma'
  Index:  (last_name, first_name, age)

  The index already contains last_name (filter), first_name, and age (selected).
  The DB reads only the index — never touches the main table rows.
  This is called an "index-only scan" — fastest possible read path.
```
