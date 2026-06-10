# How OS Manages Free Space

## Context: The Free Space Problem

- *When processes are loaded and unloaded dynamically, free memory becomes fragmented across RAM.*
- The OS must track which regions of RAM are currently free so it can satisfy future allocation requests efficiently.
- The question is: **what data structure does the OS use to store information about free memory chunks?**

---

## How the OS Tracks Free Space: The Free List

The OS maintains a **free list** — a **linked list** where each node represents one contiguous free chunk in RAM.

- Implementation is a linked list (can be singly or doubly linked).
- Each node stores the **starting address** of the free chunk and its **size**.
- The OS **iterates** over this linked list to find a suitable free chunk when a new allocation request arrives.

### Worked Example: Free List Evolution

**Setup:** Total RAM = 16KB. OS occupies addresses 0–4KB. Remaining 12KB is free initially.

**Initial state:**

```
RAM:       [OS: 0–4KB][          free: 4KB–16KB          ]
Free List: [ Start=4KB, Size=12KB ] → NULL
```

**P1 (2KB) allocated:**

```
RAM:       [OS][P1: 4–6KB][       free: 6–16KB       ]
Free List: [ Start=6KB, Size=10KB ] → NULL
```

**P2 (8KB) allocated:**

```
RAM:       [OS][P1][P2: 6–14KB][ free: 14–16KB ]
Free List: [ Start=14KB, Size=2KB ] → NULL
```

**P3 (2KB) allocated:**

```
RAM:       [OS][P1][P2][P3: 14–16KB]
Free List: NULL   ← nothing free; RAM is fully occupied
```

**P1 and P3 exit (P2 remains):**

```
RAM:       [OS][ FREE: 4–6KB ][P2: 6–14KB][ FREE: 14–16KB ]
Free List: [ Start=4KB, Size=2KB ] → [ Start=14KB, Size=2KB ] → NULL
```

- Two separate nodes because the two free chunks are **not adjacent** (P2 sits between them).
- The OS creates two distinct nodes in the linked list — one per contiguous free region.

**If P3 also exits later** — the OS checks that the two previously separate free chunks (14–16KB) and P3's region (which was also adjacent to P2's end) are now contiguous. The OS **merges** adjacent free chunks into a single larger node.

---

## External Fragmentation

**Scenario:** P3 is still in memory (8KB). P1 and P2 have exited, leaving two 2KB free chunks.

```
RAM:       [OS][ FREE 2KB ][ P3 (8KB) ][ FREE 2KB ]
Free List: [ 2KB ] → [ 2KB ] → NULL
```

A new process P4 arrives and requests 3KB.

- The OS walks the free list: first node is 2KB — too small. Second node is 2KB — too small. List ends (NULL). **Cannot satisfy the request.**
- **Total free = 4KB. Request = 3KB. Yet the request fails.**
- *This is **External Fragmentation**: the total available free memory is sufficient, but no single contiguous chunk is large enough.*

---

## Defragmentation / Compaction

To resolve external fragmentation, the OS applies **compaction** (also called **defragmentation**).

**What compaction does:** The OS physically moves the data of live processes to eliminate holes, consolidating all free space into one large contiguous block.

- *The OS monitors fragmentation continuously. When fragmentation crosses a certain threshold (too many small holes), it triggers compaction automatically — a kernel-level operation runs to rearrange memory.*

### How Compaction Works — Step by Step

**State before compaction:**

```
RAM:       [OS][ FREE 2KB ][ P3 (8KB) ][ FREE 2KB ]
           ↑              ↑            ↑
         addr 4          addr 6      addr 14
```

P3 currently starts at physical address 6. The OS shifts P3's physical data 2KB downward (toward address 4):

**State after compaction:**

```
RAM:       [OS][ P3 (8KB) ][ FREE 4KB ]
           ↑   ↑           ↑
         addr 4 addr 4    addr 12
```

- The **physical data** of P3 is moved to start at address 4 instead of 6.
- The two scattered 2KB free chunks merge into one 4KB contiguous free block at the end.

**Does P3's process break?** No — because P3 uses **logical addresses**, not physical ones. From P3's perspective, its logical address space is still 0 to 8KB. Only the underlying physical mapping changed.

- The OS simply **updates P3's Relocation Register** from its old base (6) to its new base (4).
- Everything else about P3's execution continues unchanged.
- *This is the power of the logical/physical address abstraction — the OS can move a process in physical memory at any time without the process knowing.*

**Free list after compaction:**

```
Free List: [ Start=12KB, Size=4KB ] → NULL
```

Now P4 (3KB) can be allocated from this 4KB hole:

```
RAM:       [OS][ P3 (8KB) ][ P4 (3KB) ][ FREE 1KB ]
Free List: [ Start=15KB, Size=1KB ] → NULL
```

### Why Compaction is Expensive (Limitation)

- *Compaction is itself a time-consuming process.* On a modern 16GB RAM system with many small scattered holes, moving gigabytes of live data around takes significant CPU time.
- **While compaction is running, the CPU is occupied doing this bookkeeping — user processes are paused or slowed down.**
- This degrades overall system efficiency.
- Every cycle of allocation and deallocation eventually triggers a compaction run — this is a **recurring overhead** built into the contiguous allocation model.

> **Real-world analogy:** Windows Disk Defragmenter — when you right-click a drive (e.g., C:) → Properties → Tools → Optimize, Windows applies the same defragmentation logic to the hard disk that the OS applies to RAM. Over time, as files are created and deleted, disk space fragments just like RAM. Defragmenting consolidates free space and speeds up disk access. The same concept applies to RAM memory management.

---

## Allocation Algorithms: How to Pick a Free Hole

Once the OS has a free list, when a new process arrives requesting n KB, the OS must decide **which free hole to use**. There are four standard algorithms for this.

**Example free list (used for all four algorithms below):**

```
[ 100KB ] → [ 90KB ] → [ 50KB ] → [ 200KB ] → NULL
```

**Request:** a new process needs **90KB**.

---

### 1. First Fit

**Rule:** Scan the free list from the beginning and allocate from the **first hole that is large enough** to satisfy the request.

- No comparison across holes — take the first one that fits.
- Stop as soon as a suitable hole is found.

**Trace (request = 90KB):**

```
Check 100KB → 100 ≥ 90 → YES → allocate here
```

**Result:**

```
Free list after: [ 10KB ] → [ 90KB ] → [ 50KB ] → [ 200KB ] → NULL
```

- 90KB is taken from the 100KB hole, leaving a 10KB remainder in its place.

| Property | Detail |
|---|---|
| Speed | **Fast** — stops at the first suitable hole; no full traversal in best case |
| Simplicity | **Easy to implement** — straightforward linear scan |
| Worst-case complexity | O(n) — if the suitable hole is at the end of the list |

---

### 2. Next Fit

**Rule:** Same as First Fit, but instead of always starting from the beginning of the list, it **starts from where the last allocation left off**.

- The OS saves a pointer to the last allocated position.
- On the next request, scanning resumes from that saved position (not from the head of the list).

**Trace (two consecutive requests = 90KB then 10KB):**

- Request 1 (90KB): scan from head → 100KB found → allocate → pointer saved at this position.
- Request 2 (10KB): scan starts from *after* the previous allocation point → next node is 90KB → 90 ≥ 10 → allocate here.

| Property | Detail |
|---|---|
| Speed | **Fast** — similar to First Fit |
| Difference from First Fit | Avoids repeatedly scanning the beginning of the list; distributes allocations more evenly across the list |

---

### 3. Best Fit

**Rule:** Scan the **entire** free list and allocate from the **smallest hole that is still large enough** to satisfy the request.

- This minimizes the leftover (remainder) fragment inside the chosen hole.
- Requires a full traversal of the list every time.

**Trace (request = 90KB):**

```
Check 100KB → fits, remainder = 10KB
Check  90KB → fits, remainder =  0KB  ← smallest remainder so far
Check  50KB → does not fit
Check 200KB → fits, remainder = 110KB
Best choice: 90KB hole (smallest remainder = 0)
```

**Result:**

```
Free list after: [ 100KB ] → [ 50KB ] → [ 200KB ] → NULL
(90KB hole is fully consumed — removed from list)
```

| Property | Detail |
|---|---|
| Internal remainder | **Minimum** — chosen hole wastes the least space |
| Speed | **Slower** — must always traverse the entire list |
| Major drawback | *Causes severe external fragmentation over time* |

**Why Best Fit leads to external fragmentation:**
- Best Fit leaves behind the smallest possible remainders — fragments like 5KB, 3KB, 2KB, 1KB.
- Over many allocations, the free list fills with these tiny fragments.
- Eventually, even though total free space adds up to, say, 50KB, no individual hole is large enough for a 7KB request — because every hole is 5KB or smaller.
- *Best Fit paradoxically creates the most external fragmentation by trying hardest to minimize internal waste.*

---

### 4. Worst Fit

**Rule:** Scan the **entire** free list and allocate from the **largest hole available**.

- This intentionally maximizes the leftover fragment — the idea is that the remainder left behind is large enough to still be useful for future allocations.
- Requires a full traversal of the list every time.

**Trace (request = 90KB):**

```
Check 100KB → fits, remainder = 10KB
Check  90KB → fits, remainder =  0KB
Check  50KB → does not fit
Check 200KB → fits, remainder = 110KB  ← largest remainder
Best choice: 200KB hole (largest hole)
```

**Result:**

```
Free list after: [ 100KB ] → [ 90KB ] → [ 50KB ] → [ 110KB ] → NULL
```

- 90KB allocated from 200KB, leaving a 110KB remainder — still a large, usable chunk.

| Property | Detail |
|---|---|
| Leftover fragment | **Maximum** — deliberately leaves large remainders |
| Speed | **Slower** — must always traverse the entire list |
| Advantage | *Less external fragmentation* than Best Fit — large remainders can satisfy future requests |
| Rationale | Greedy: always pick the biggest hole so the leftover is as large and reusable as possible |

---

### Comparison Table

| Algorithm | Traversal | Speed | Internal Fragmentation (per hole) | External Fragmentation (over time) |
|---|---|---|---|---|
| First Fit | Partial (stops at first fit) | Fast | Moderate | Moderate |
| Next Fit | Partial (from last position) | Fast | Moderate | Moderate |
| Best Fit | Full list | Slow | **Minimum** | **Maximum** (tiny leftover holes) |
| Worst Fit | Full list | Slow | Maximum | **Minimum** (large leftover holes) |

> *All four algorithms apply to contiguous memory allocation. When contiguous allocation's fragmentation problems become too severe, the OS switches to non-contiguous techniques — paging and segmentation — covered in the next lectures.*
