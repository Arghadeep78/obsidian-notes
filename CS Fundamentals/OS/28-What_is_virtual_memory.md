# **Virtual Memory**

---

## **The Problem: Limited Physical Memory**

- We have two kinds of memory: **physical memory** (RAM, with physical address space) and **logical memory** (logical address space, one per process)
- In paging, every page of a process must be allocated a frame in RAM before it can execute
- Consider a concrete scenario:
  - RAM = 12 KB, page size = 1 KB → 12 frames total
  - P1 = 6 KB → needs 6 frames
  - P2 = 5 KB → needs 5 frames
  - After allocating P1 and P2: 11 frames used, only 1 frame free
  - P3 = 5 KB arrives → needs 5 frames → **cannot be loaded** — only 1 frame is free

- With traditional full-process loading: only 2 processes can run in a 12 KB RAM
- On a modern PC with 16 GB RAM, GTA 5 (which is 30+ GB) still runs — *this is virtual memory making that possible*
- Instructions must be in physical memory to be executed — but this limits the size of a program to the size of physical memory. In many cases, the entire program is not needed at the same time. Executing a program that is only partially in memory gives many benefits:
  - A program would no longer be constrained by the amount of physical memory available.
  - Each user program takes less physical memory → more programs can run at the same time → increase in CPU utilization and throughput.
  - Running a program not entirely in memory benefits both the system and the user.
- The programmer is provided a very large virtual memory even when only a smaller physical memory is available.

---

## **The Core Idea: Load Only Needed Pages**

- At any point during execution, **not all pages of a process are in active use simultaneously**
- Example: A process may have code for many switch-case branches, error handlers, or rarely-visited paths — those pages may never be needed during a particular run
- Real-world analogy: In GTA 5, if you are in the north-west area of the map, the OS only needs to keep pages for that area in RAM. The south-west area's data never needs to be loaded if you never go there.

**Observation:** We only need to load the pages that are *currently needed*. The rest can be kept elsewhere.

**Revised allocation with only needed pages:**

| Process | Total size | Needed pages (example) | Frames used |
|---------|-----------|------------------------|-------------|
| P1 | 6 KB | 3 pages needed | 3 frames |
| P2 | 5 KB | 1 page needed | 1 frame |
| P3 | 5 KB | 2 pages needed | 2 frames |
| **Total** | **16 KB** | | **6 frames** |

- Previously only P1 + P2 could fit; now P1 + P2 + P3 all run using only 6 of 12 frames
- The remaining 6 frames can be used for P4 or other processes
- *Total process sizes (16 KB+) exceed physical RAM (12 KB), yet all processes can run*

---

## **Swap Space**

- The OS reserves a dedicated region inside the hard disk called the ***swap space*** (also called *backing store*)
- Disk is vastly larger than RAM (e.g., 1 TB disk vs. 16 GB RAM), so reserving even 1 GB of swap space costs nothing meaningful to the user
- This reserved area is **not accessible to user programs** — it is exclusively managed by the OS

**How it works:**
- *Needed pages* of a process → loaded into actual RAM frames
- *Not-needed pages* of a process → stored in swap space on disk, ready to be brought in on demand

```css
+------------------+          +-----------------------------+
|   Physical RAM   |          |         Hard Disk           |
|  (12 KB)         |          |  +----------------------+   |
|                  |  swap    |  |      Swap Space       |   |
| [P1: page 0]     | <------> |  | P1: page 1, page 3.. |   |
| [P1: page 2]     |   in/out |  | P2: page 1, page 2.. |   |
| [P2: page 0]     |          |  | P3: page 0, page 2.. |   |
| [P3: page 1]     |          |  +----------------------+   |
|   ...            |          |   (OS-reserved region)      |
+------------------+          +-----------------------------+
  Actual physical                    Secondary memory
  memory (fast)                      (slow, but large)
```

**Swap In:** Moving a page *from swap space → RAM* (when the CPU needs that page)
**Swap Out:** Moving a page *from RAM → swap space* (when that page is no longer immediately needed, to free a frame)

---

## **The Illusion: What Virtual Memory Actually Is**

The OS tricks the user — and the CPU — into believing that *all pages of every process are already in memory*.

- **User/process perspective:** "My entire program is in RAM, everything is accessible"
- **Reality:** Only *needed pages* are in actual RAM. The rest sit in swap space.
- The OS silently moves pages in and out of RAM as needed, behind the scenes. The user notices nothing.

> **Virtual Memory = RAM + Swap Space (combined)**
>
> *The RAM + swap space arrangement together creates the illusion of one large memory. Since it is not entirely real (RAM) and only partially exists physically, this combined "memory" is called **virtual memory**.*

**Concrete example:**
- RAM = 12 KB (physical)
- Running processes: P1(6 KB) + P2(5 KB) + P3(5 KB) + P4(3 KB) = **19 KB total**
- 19 KB of processes running inside 12 KB of RAM — this is possible because only the needed pages of each process occupy RAM at any moment; the rest are in swap space
- From the user's perspective: the system appears to have more than 19 KB of memory available

---

## **Definition**

> *Virtual memory is a technique that allows the execution of processes that are **not completely** loaded in memory. It provides the user an **illusion of having a very large main memory**. This is achieved by treating a reserved part of secondary memory (the swap space) as an extension of main memory.*

---

## **Advantages of Virtual Memory**

**1. Programs can be larger than physical memory**
- A single 20 KB process can run in a 12 KB RAM: only needed pages (say, 4–5 pages) are loaded; the remaining 15+ pages stay in swap space and are brought in as needed.

**2. Degree of multiprogramming increases**
- Instead of loading full processes, each process occupies only a few frames → more processes can coexist in RAM simultaneously → CPU utilization improves

**3. Users can run large applications with less real memory**
- GTA 5 is 30 GB; your RAM is 16 GB. Only the pages the game actively needs at any moment (the current map area, the loaded assets) are in RAM. Everything else is in swap space or on the game's disk files.

**Disadvantages:**

- **System can become slower due to swapping overhead:** Disk access (swap space) is much slower than RAM. Every swap-in or swap-out takes time. If too many swaps happen, the system feels sluggish.
- **Risk of thrashing:** If the OS loads too many processes with too few pages each, the CPU ends up spending most of its time handling page faults rather than executing actual code. This is called *thrashing* and is covered in lecture 31.

---

## **Demand Paging**

*Demand paging is the most popular method to implement virtual memory.*

**Core rule:** *Never bring a page into memory unless it is demanded (needed) by the CPU.*

- Pages of a process that are not currently needed are stored in secondary memory (swap space)
- A page is copied into RAM **only when the CPU demands it** — i.e., when execution reaches an instruction that references that page
- This is called ***on-demand loading***

### **Pager vs. Swapper — Important Distinction**

Both "swapper" and "pager" move process data between RAM and swap space, but they operate at different granularities:

| Term | What it moves | When used |
|------|--------------|-----------|
| **Swapper** | The *entire process* (all pages) | Medium-Term Scheduler — when degree of multiprogramming is too high, the entire process is swapped out to reduce load |
| **Pager** | *Individual pages* of a process | Demand paging — only the needed pages of a process are swapped in/out |

In virtual memory / demand paging, we use the term ***pager*** — because we are managing individual pages, not entire processes.

**Why "lazy pager"?**
- The pager is called *lazy* because it does the minimum work: it never preloads pages that might not be needed
- When a process is to be swapped in, the pager guesses which pages will be used and brings only those into memory — avoiding reading pages that will not be used anyway
- At process startup: the pager loads only the page containing `main()` (the entry point) and perhaps a few nearby pages
- All other pages go to swap space initially
- As the program runs and needs new pages, the pager brings them in *on demand*
- This way, the OS decreases the swap time and the amount of physical memory needed

---

## **Valid-Invalid Bit**

To track whether a page is currently in RAM or in swap space, the page table has an extra column: the ***valid-invalid bit***.

| Bit Value | Meaning |
|-----------|---------|
| **1 (valid)** | The page is **in physical RAM** at the frame number listed in the page table. The address is legal (within this process's logical address space). |
| **0 (invalid)** | Either: (a) the page is **in swap space** (not currently in RAM), or (b) the logical address is **illegal** — it does not belong to this process at all |

**Example — a process with 8 pages (A through H), page table:**

```css
Logical Page | Frame # | Valid-Invalid Bit | Location
-------------|---------|-------------------|--------------------
     0 (A)   |    4    |        1          | In RAM, frame 4
     1 (B)   |    -    |        0          | In swap space
     2 (C)   |    6    |        1          | In RAM, frame 6
     3 (D)   |    -    |        0          | In swap space
     4 (E)   |    -    |        0          | In swap space
     5 (F)   |    9    |        1          | In RAM, frame 9
     6 (G)   |    -    |        0          | In swap space
     7 (H)   |    -    |        0          | In swap space
```

- Pages A, C, F are currently in RAM (valid bit = 1) → can be accessed directly
- Pages B, D, E, G, H are in swap space (valid bit = 0) → accessing any of them triggers a **page fault**

---

## **Page Fault**

*A page fault is an interrupt (trap/exception) generated when the CPU tries to access a page whose valid-invalid bit is 0 — meaning the page is not currently in RAM.*

Page fault ≠ error. It is a normal, expected event in a virtual memory system. The OS handles it and brings the page in.

### **Page Fault Handling — Step by Step**

```css
Step 1: CPU generates a logical address and checks the page table
              |
              v
Step 2: Valid-Invalid bit = 0 → page not in RAM
        → CPU generates a TRAP (page fault exception)
        → Control transfers to the OS
              |
              v
Step 3: OS checks — is this a valid address?
        ├─ If INVALID (address not in this process's logical address space)
        │     → OS terminates the process (illegal memory access)
        │
        └─ If VALID (address belongs to this process, page is in swap space)
              |
              v
Step 4: OS locates the page in backing store (swap space) on disk
              |
              v
Step 5: OS looks for a free frame in RAM (from the free frame list)
        ├─ If a free frame exists → use it
        └─ If NO free frame exists → run page replacement algorithm
              to select a victim page, swap it out, free that frame
              |
              v
Step 6: Schedule disk I/O: read the desired page from swap space
        into the now-free frame in RAM
              |
              v
Step 7: Update the page table:
        - Set the frame number for this page
        - Set valid-invalid bit to 1
              |
              v
Step 8: Re-execute the instruction that caused the page fault
        (now the page is in RAM, execution continues normally)
```

- **Page Fault Service Time** = total time taken to handle one page fault (mainly disk I/O time, which is slow)
- *Fewer page faults → less page fault service time → better system performance*

---

## **Pure Demand Paging**

Pure demand paging is an *extreme version* of demand paging:

- **At process startup: zero pages are loaded into RAM** — the entire process goes into swap space first
- When the OS sets the instruction pointer to the first instruction of the process, the page containing that instruction is not in RAM → immediate page fault → page is loaded
- Every new page needed causes a page fault until execution stabilizes

**Problem with pure demand paging:**
- The startup phase generates a flood of page faults (one per new page touched)
- Each page fault requires a disk access — which is slow
- This makes program startup very costly

**Practical approach — Locality of Reference:**

Instead of pure demand paging, the pager uses the ***Locality of Reference*** principle:

- Programs tend to execute in *localities* — clusters of pages used together during a phase of execution
- At startup, the pager loads not just `main()` but also the pages of functions that `main()` immediately calls, and surrounding code
- This *pre-loads the current working locality*, drastically reducing initial page faults
- When execution moves to a new locality (e.g., a different function call chain), a small burst of page faults occurs, and the new locality is loaded

> *Locality of Reference:* At any point in time, a program references only a small subset of its pages — those in its current locality. Pages outside the locality are unlikely to be needed immediately.

This is also the basis of the *Working Set Model* (covered in lecture 31).

---

## **Summary of the Full Picture**

```css
Process's full logical address space
           |
           | (split by pager using locality of reference)
           |
    +------+------+
    |             |
  Needed pages  Not-needed pages
  (in RAM)      (in swap space on disk)
    |             |
    |   page fault triggers swap-in when needed
    +<────────────+
```

- The OS creates an illusion: the process behaves as if all its pages are in a large memory
- Internally: only a small working set of pages is in RAM at any time; the rest are in swap space
- The valid-invalid bit in the page table is the mechanism that triggers this on-demand loading
- Page faults are the events that drive the system to swap pages in/out
- Page replacement algorithms (lecture 30) decide *which page to evict* when RAM is full and a new page must be loaded
