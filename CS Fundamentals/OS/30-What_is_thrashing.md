# **Thrashing**

---

## **Why Thrashing Matters as an Interview Topic**

When an interviewer asks "What is thrashing?", they are not just asking for a one-line definition. A complete answer requires understanding:
- *Paging* — how processes are divided into pages and loaded into frames
- *Virtual memory / demand paging* — only needed pages are in RAM; the rest are in swap space
- *Page faults* — what happens when a needed page is not in RAM
- *Page fault service time* — the overhead cost of handling a page fault (disk I/O)
- *Page replacement algorithms* — how the OS decides which page to evict
- *Degree of multiprogramming* — how many processes are simultaneously in memory

*Thrashing is the natural endpoint when all these concepts interact poorly.*

*Prerequisite: Lectures 26–29 (paging, virtual memory, demand paging, page faults)*

---

## **Setup: Two Situations to Compare**

Consider a RAM with 6 frames. Two different ways to allocate those frames:

**Situation 1 — Few processes, each with several pages:**
```css
Frame:  [ P1:pg1 ] [ P1:pg2 ] [ P1:pg3 ]
        [ P2:pg1 ] [ P2:pg2 ] [ P2:pg3 ]
```
- 2 processes in RAM
- Each process has 3 pages loaded
- Degree of multiprogramming = 2 (low)

**Situation 2 — Many processes, each with only one page:**
```css
Frame:  [ P1:pg1 ] [ P2:pg1 ] [ P3:pg1 ]
        [ P4:pg1 ] [ P5:pg1 ] [ P6:pg1 ]
```
- 6 processes in RAM
- Each process has only 1 page loaded
- Degree of multiprogramming = 6 (high)

**Question:** Which situation has more page faults?

**Answer: Situation 2** — despite its higher degree of multiprogramming.

---

## **Why Situation 2 Causes More Page Faults**

Recall: a **page fault** occurs when the CPU needs a page that is not currently in RAM. Handling it means going to disk (swap space), which is slow — this is the *page fault service time* overhead.

**In Situation 2 (one page per process):**

1. P1 starts executing its single loaded page
2. Almost immediately, it needs page 2 → **page fault** (page 2 is in swap space, not in RAM)
3. To load P1's page 2, some existing page must be evicted (since all 6 frames are full)
4. Say P5's page 1 is evicted (swapped out) and P1's page 2 is swapped in
5. Now if P3 runs, it finishes its single page and needs page 2 → **page fault again**
6. To load P3's page 2, another page must be evicted — say P6's page 1 is swapped out
7. Shortly after, P5 or P6 needs to run again → **page fault** (their page was just evicted)
8. The page just loaded to handle P5's fault must evict some other process's only page → triggering yet another page fault shortly after

**The cycle:**
```css
Process runs → needs next page → page fault
→ evict another process's only page → load needed page
→ that other process runs → needs its evicted page → page fault
→ evict another process's page → ...
→ INFINITE PAGE FAULT LOOP
```

*The CPU spends nearly all its time servicing page faults. Almost no useful computation happens. This is **thrashing**.*

**In Situation 1 (three pages per process):**
- Each process has enough pages loaded to run for a meaningful period before needing a new page
- Page faults still occur (when execution moves to a new page), but much less frequently
- CPU actually executes process code between page faults
- Result: higher actual throughput despite lower degree of multiprogramming

---

## **Definition of Thrashing**

> *When a process does not have enough frames to support its currently active pages, it will rapidly generate page faults. Each page fault causes a page to be swapped in from disk, which requires evicting some other page. If the evicted page is also actively needed, it immediately generates another page fault when accessed. This cycle of constant swap-in and swap-out is called **thrashing**.*
>
> ***Thrashing = a state where the system spends more time servicing page faults than executing processes.***

- Characterized by **high paging activity** — rapid, repeated swap-in / swap-out
- During thrashing: *no useful work* is being done — CPU cycles are consumed entirely by page fault handling
- CPU utilization drops sharply even though many processes are nominally "active" in RAM

---

## **CPU Utilization vs. Degree of Multiprogramming**

There is a well-defined relationship between how many processes are in RAM (degree of multiprogramming) and how effectively the CPU is used (CPU utilization):

```css
CPU Utilization
     ^
     |                  * ← peak: thrashing threshold
     |               *     *
     |            *           *
     |          *               *
     |       *                    * * * ← thrashing zone (CPU utilization collapses)
     |    *
     |  *
     | *
     +--------------------------------------------> Degree of Multiprogramming
      low                                     high
```

### **Explaining each region:**

**Low degree of multiprogramming → low CPU utilization:**
- Example: only 2 processes are in RAM
- If both processes go to I/O simultaneously (waiting for disk, keyboard, etc.), the CPU has nothing to execute → sits idle
- CPU utilization is low not because of page faults, but because of insufficient concurrency

**Increasing degree of multiprogramming → increasing CPU utilization:**
- As more processes are loaded, there is always some process ready to run even when others are doing I/O
- CPU stays busy more of the time
- This is the desired operating range

**Beyond the peak → CPU utilization drops sharply (thrashing begins):**
- Too many processes are in RAM, each with too few frames
- Every process generates page faults rapidly
- Each page fault handling evicts a page from another process
- That other process then immediately generates a page fault
- The CPU spends most of its time waiting for disk I/O (page fault service) rather than executing code
- CPU utilization collapses — the CPU looks idle, but it is actually just waiting for disk constantly

---

## **How Thrashing Escalates — The OS Trap**

The OS scheduler monitors CPU utilization. When it drops, the OS responds by loading more processes to increase multiprogramming. But if the drop is caused by thrashing, this makes things worse:

```css
Step 1: CPU utilization drops
         ↓
Step 2: OS scheduler sees low CPU utilization
        → OS (Long-Term Scheduler) loads more processes into RAM
         ↓
Step 3: More processes → each process gets fewer frames
        (a global page replacement algorithm evicts pages without
         regard for which process owns them, so every new process
         steals frames from existing processes)
         ↓
Step 4: Fewer frames per process → more page faults per process
         ↓
Step 5: More page faults → more disk I/O → CPU spends more time
        waiting for disk, less time on actual computation
         ↓
Step 6: CPU utilization drops further
         ↓
Step 7: OS sees low CPU utilization again → loads MORE processes
         ↓
        [Back to Step 3 — cycle worsens]
         ↓
RESULT: System is in thrashing — swap-in swap-out consuming
        everything, no useful work being done
```

*The key cause:* a **global page replacement algorithm** evicts pages without distinguishing which process they belong to. When a new process is loaded, it takes frames from whichever processes the replacement algorithm selects — leaving other processes with fewer frames than they need, causing their page fault rates to spike.

---

## **Handling and Preventing Thrashing**

### **Method 1: Working Set Model**

**Based on the principle of locality:**

A process does not access all its pages randomly. It works in *localities* — during a phase of execution, it repeatedly references a small set of pages (the current working set). After that phase, it transitions to a new locality.

- Example: `main()` calls functions A, B, C → the pages containing `main`, A, B, C form the current locality. For a period, only those pages are actively needed.
- When execution shifts to a new call chain, a new locality forms.

**Strategy:**
- Determine the process's **current locality** (the set of pages actively in use right now)
- Allocate *at least enough frames to hold the entire current locality*

**Formal statement:**

> *If we allocate enough frames to a process to accommodate its current locality, it will only page-fault when it moves to a new locality — not within a locality. This prevents thrashing.*

**What happens if frames < current locality size:**
- The process cannot hold all its active pages in RAM simultaneously
- It constantly page-faults, evicting pages it will need again moments later
- → The process thrashes

**What happens if frames ≥ current locality size:**
- All currently active pages fit in RAM
- Page faults only occur at locality transitions (when the process shifts to a new phase)
- The process runs efficiently between transitions

---

### **Method 2: Page Fault Frequency (PFF)**

Instead of predicting localities in advance, monitor the page fault rate in real time and use it as a signal to adjust frame allocation.

**Core idea:** Use the page fault rate as a tuning knob.
- *Too many page faults* → the process needs more frames
- *Too few page faults* → the process has more frames than it needs; reclaim some for other processes

**Set an upper and lower bound on the acceptable page fault rate:**

```css
Page Fault Rate (for a given process)
     ^
     |
   --+------ Upper Bound ──────────────────────  ← fault rate too high
     |
     |         [  Acceptable Zone  ]               ← target operating range
     |
   --+------ Lower Bound ──────────────────────  ← fault rate too low
     |
     +---------------------------------------------> Number of Frames
                                                     allocated to the process
```

**Note on the axes:** As the number of frames allocated to a process *increases*, its page fault rate *decreases* (more pages fit in RAM → fewer faults). The graph slopes downward from left to right.

**Rule: if page fault rate > upper bound:**
- This process is generating too many page faults → it does not have enough frames for its current locality
- **Action:** Allocate additional frames to this process (bring more of its pages into RAM)
- If there are no free frames available in the system → use the Medium-Term Scheduler to **swap out an entire process** (not just pages) — freeing a large block of frames — and distribute those frames to the processes that need them

**Rule: if page fault rate < lower bound:**
- This process is generating very few page faults → it has more frames than it actually needs (perhaps most of its pages are loaded, including ones it rarely uses)
- **Action:** **Reclaim frames** from this process — move some of its less-used pages back to swap space, freeing those physical frames
- Give the reclaimed frames to other processes that have high page fault rates, or use them to increase the degree of multiprogramming by loading a new process

**Goal:** Keep every process's page fault rate within the acceptable zone — avoiding both thrashing (too high) and inefficient frame hoarding (too low). By controlling the page fault rate this way, thrashing can be prevented.

---

## **The Complete Picture: How Everything Connects**

```css
Paging (divide process into fixed-size pages; load into frames)
       ↓
Virtual Memory + Demand Paging
(only load needed pages; rest in swap space; creates illusion of large memory)
       ↓
Page Faults
(occur when a needed page is not in RAM; each fault costs disk access time)
       ↓
Page Replacement Algorithms
(decide which page to evict when a new page must be loaded; goal: minimize faults)
       ↓
Degree of Multiprogramming
(how many processes are in RAM simultaneously; impacts CPU utilization)
       ↓
Thrashing
(too many processes → too few frames each → excessive page faults → CPU does no useful work)
       ↓
Solutions
(Working Set Model: allocate enough frames for current locality;
 Page Fault Frequency: monitor fault rate, adjust frames dynamically)
```

**The fundamental trade-off:**

| Situation | Frames per process | Page fault rate | CPU utilization |
|-----------|-------------------|-----------------|-----------------|
| Too few processes in RAM | High | Low | Low (CPU idles during I/O) |
| Optimal number of processes | Moderate | Moderate | **High** ← target |
| Too many processes in RAM | Too low | Very high (thrashing) | Low (CPU stuck on page faults) |

**The real hardware fix:** Add more physical RAM. More RAM means more frames → each process can hold more pages → fewer page faults → higher multiprogramming without thrashing. Since RAM is expensive, virtual memory + the working set model + PFF are the OS's software-level tools to extract maximum performance from limited physical memory.

---

## **Key Takeaways**

- *Thrashing* = the system spends more time servicing page faults (swap-in/swap-out disk operations) than actually executing process code
- It occurs when processes have too few frames to hold their active working pages — causing a rapid cycle of eviction and re-loading of the same pages
- The CPU utilization vs. degree-of-multiprogramming curve rises to a peak, then drops sharply — the drop marks the onset of thrashing
- The OS scheduler can make thrashing worse by interpreting low CPU utilization as a signal to load more processes — which reduces frames per process further
- **Working Set Model:** prevent thrashing by ensuring each process always has enough frames for its current locality
- **Page Fault Frequency:** detect and respond to thrashing dynamically — allocate more frames when fault rate is too high, reclaim frames when fault rate is too low
