# **Page Replacement Algorithms**

---

## **Recap: Why Page Replacement Is Needed**

In demand paging (virtual memory), RAM is shared among multiple processes. Each process has only some of its pages loaded in RAM at any time; the rest are in swap space.

**The page fault scenario:**
- A running process needs a page (e.g., P1's page 3) that is currently in swap space, not in RAM
- If a **free frame exists** in RAM → load the needed page directly into that frame
- If **no free frame exists** (all frames are occupied) → the OS must **evict one existing page** from RAM to swap space, freeing a frame, then load the needed page

*The decision of which page to evict is made by a **page replacement algorithm**.*

> **Goal of every page replacement algorithm: minimize the number of page faults.**
>
> Each page fault incurs a *page fault service time* — the time to move a page from disk (swap space) to RAM. Disk access is slow. Fewer page faults = less overhead = faster system.

---

## **Reference String**

A *reference string* is the sequence of page numbers requested by the CPU over time. It is used to evaluate and compare page replacement algorithms by counting how many page faults each algorithm produces for the same sequence.

**Reference string used in all examples below:**
```css
7, 0, 1, 2, 0, 3, 0, 4, 2, 3, 0, 3, 2, 1, 2, 0, 1, 7, 0, 1
```

**Assumption:** 3 frames are available (initially empty). The rest of RAM is hypothetically occupied by other processes.

**Two possible outcomes on each access:**
- **Page Hit:** The requested page is already in RAM → no page fault, no disk access
- **Page Miss (Page Fault):** The requested page is not in RAM → page fault occurs, disk access required

---

## **1. FIFO — First In, First Out**

### **Rule**
When all frames are full and a new page must be loaded, **evict the page that has been in RAM the longest** (the one that arrived first).

### **Walkthrough**

Starting state: all 3 frames empty.

```css
Step  | Page | Frame 1 | Frame 2 | Frame 3 | Event
------|------|---------|---------|---------|------------------
  1   |  7   |   7     |   -     |   -     | Page fault #1  (7 loaded, frame was empty)
  2   |  0   |   7     |   0     |   -     | Page fault #2  (0 loaded, frame was empty)
  3   |  1   |   7     |   0     |   1     | Page fault #3  (1 loaded, frame was empty)
  4   |  2   |   2     |   0     |   1     | Page fault #4  (all frames full; 7 is oldest → evict 7, load 2)
  5   |  0   |   2     |   0     |   1     | Page HIT       (0 already in RAM)
  6   |  3   |   2     |   3     |   1     | Page fault #5  (0 is now oldest → evict 0, load 3)
  7   |  0   |   2     |   3     |   0     | Page fault #6  (1 is now oldest → evict 1, load 0)
  8   |  4   |   4     |   3     |   0     | Page fault #7  (2 is now oldest → evict 2, load 4)
  9   |  2   |   4     |   2     |   0     | Page fault #8  (3 is now oldest → evict 3, load 2)
  10  |  3   |   4     |   2     |   3     | Page fault #9  (0 is now oldest → evict 0, load 3)
  11  |  0   |   0     |   2     |   3     | Page fault #10 (4 is now oldest → evict 4, load 0)
  12  |  3   |   0     |   2     |   3     | Page HIT       (3 already in RAM)
  13  |  2   |   0     |   2     |   3     | Page HIT       (2 already in RAM)
  14  |  1   |   0     |   1     |   3     | Page fault #11 (2 is now oldest → evict 2, load 1)
  15  |  2   |   0     |   1     |   2     | Page fault #12 (3 is now oldest → evict 3, load 2)
  16  |  0   |   0     |   1     |   2     | Page HIT       (0 already in RAM)
  17  |  1   |   0     |   1     |   2     | Page HIT       (1 already in RAM)
  18  |  7   |   7     |   1     |   2     | Page fault #13 (0 is now oldest → evict 0, load 7)
  19  |  0   |   7     |   0     |   2     | Page fault #14 (1 is now oldest → evict 1, load 0)
  20  |  1   |   7     |   0     |   1     | Page fault #15 (2 is now oldest → evict 2, load 1)
```

**Total page faults: 15**

### **Pros**
- Easy to implement — just maintain the order in which pages entered RAM (a queue)

### **Cons**
- Performance is **not always good** — it can make poor eviction choices
- Two specific failure cases:
  1. **Evicting a heavily-used page:** A page loaded at startup and referenced frequently later gets evicted just because it is "old." Example: page 7 is evicted at step 4 but is referenced again at step 18 — causing another page fault.
  2. **Keeping a page that is no longer needed:** A page that was used only once at the beginning (e.g., an initialization module) stays in RAM until it's the oldest, even though it will never be used again — wasting a frame.

---

### **Bélády's Anomaly**

*Expected behavior:* As the number of frames increases, page faults should **decrease or stay the same** — more room means fewer evictions.

In FIFO, this expectation is **violated for certain reference strings**: adding more frames can *increase* page faults. This is ***Bélády's Anomaly***.

**Example — reference string:** `1, 2, 3, 4, 1, 2, 5, 1, 2, 3, 4, 5`

| Number of frames | Page faults |
|-----------------|-------------|
| 2 | 12 |
| 3 | 9 |
| **4** | **10** ← *more faults than with 3 frames* |
| 5 | 5 |
| 6 | 5 |

```css
Expected (normal behavior):          FIFO (with Bélády's Anomaly):

Page faults                           Page faults
    ^                                     ^
    |  \                                  |  \
    |   \                                 |   \ /\
    |    \___                             |    v   \___
    |                                     |
    +-------> # Frames                    +-------> # Frames
              (always decreasing)                (bump at 4 frames)
```

- Bélády's Anomaly is a **known weakness of FIFO**
- In LRU and Optimal Page Replacement, increasing the number of frames **always reduces or maintains** the page fault count — it does **not** exhibit Bélády's Anomaly

---

## **2. Optimal Page Replacement (OPR)**

### **Rule**
When eviction is needed, **replace the page that will not be used for the longest time in the future** — or a page that will never be referenced again in the future.

If no such "never used again" page exists, replace the page whose *next reference is furthest away* in the future reference string.

### **Why it is the best algorithm**
- OPR produces the **minimum possible number of page faults** for any reference string and any number of frames
- No other algorithm can do better — it is the theoretical lower bound
- Used as a **reference benchmark** to evaluate how close other algorithms come to optimal

### **Why it cannot be implemented in a real OS**
- It requires **knowledge of the future** — specifically, knowing all upcoming page references in advance
- In a real OS, processes are dynamic: user input, network events, conditionals — the future reference sequence cannot be known
- *Analogous to SJF (Shortest Job First) scheduling*, which is also theoretically optimal but requires knowing burst times in advance

### **Walkthrough**

At each page fault, look ahead in the reference string to find which of the currently loaded pages will be referenced furthest in the future (or not at all), and evict that one.

```css
Step  | Page | Frames (after action) | Event & Reasoning
------|------|-----------------------|------------------------------------------
  1   |  7   | [7, -, -]             | PF#1 — empty frame
  2   |  0   | [7, 0, -]             | PF#2 — empty frame
  3   |  1   | [7, 0, 1]             | PF#3 — empty frame
  4   |  2   | [2, 0, 1]             | PF#4 — all full; look ahead:
      |      |                       |   7 next appears at position 18 (far future)
      |      |                       |   0 next appears at position 5  (soon)
      |      |                       |   1 next appears at position 14 (later)
      |      |                       |   → 7 is furthest → evict 7, load 2
  5   |  0   | [2, 0, 1]             | Hit (0 present)
  6   |  3   | [2, 0, 3]             | PF#5 — all full; look ahead:
      |      |                       |   2 next appears at position 9  (soon)
      |      |                       |   0 next appears at position 7  (soon)
      |      |                       |   1 next appears at position 14 (furthest)
      |      |                       |   → 1 is furthest → evict 1, load 3
  7   |  0   | [2, 0, 3]             | Hit
  8   |  4   | [4, 0, 3]             | PF#6 — look ahead:
      |      |                       |   2 next appears at position 9  (soon)
      |      |                       |   0 next appears at position 11 (later)
      |      |                       |   3 next appears at position 10 (soon)
      |      |                       |   → 0 is furthest → evict 0... 
      |      |                       |   Wait: 0 at 11, 3 at 10, 2 at 9
      |      |                       |   → 0 is furthest → evict 0, load 4
  ...continuing similarly...

Total page faults: 9
```

### **Result**

| Algorithm | Page Faults | Implementable? |
|-----------|-------------|----------------|
| FIFO | 15 | Yes |
| **Optimal (OPR)** | **9 (minimum possible)** | **No** |

- OPR achieves the *theoretical minimum* — no algorithm can produce fewer than 9 faults on this reference string with 3 frames
- It is **impossible to implement** in practice but is used as the ideal reference point

---

## **3. LRU — Least Recently Used**

### **Core Idea**
OPR uses *future* knowledge. LRU approximates OPR using *past* knowledge:

> *"Use the recent past as an approximation for the near future."*
>
> A page that has not been used for a long time is unlikely to be needed soon. Replace the page that was **least recently used** — the one whose most recent reference is furthest in the past.

This is justified by the *locality of reference* principle: programs tend to reuse the same pages repeatedly for a period, then shift to a new set.

### **Walkthrough**

At each step, track *when each page was last used*. On a page fault, evict the page with the oldest last-use time.

```css
Step  | Page | Frames (after) | Event & LRU reasoning
------|------|----------------|------------------------------------------
  1   |  7   | [7, -, -]      | PF#1 — empty frame; last used: 7@1
  2   |  0   | [7, 0, -]      | PF#2 — empty frame; last used: 7@1, 0@2
  3   |  1   | [7, 0, 1]      | PF#3 — empty frame; last used: 7@1, 0@2, 1@3
  4   |  2   | [2, 0, 1]      | PF#4 — all full; LRU = 7 (last used at step 1)
      |      |                |   → evict 7, load 2; last used: 0@2, 1@3, 2@4
  5   |  0   | [2, 0, 1]      | HIT — 0 accessed; update: last used: 1@3, 2@4, 0@5
  6   |  3   | [2, 3, 1]      | PF#5 — LRU = 0? No: 0@5, 2@4, 1@3 → LRU = 1 (step 3)
      |      |                |   → evict 1, load 3; last used: 2@4, 0@5, 3@6
  7   |  0   | [2, 3, 0]      | HIT — update: last used: 2@4, 3@6, 0@7
  8   |  4   | [4, 3, 0]      | PF#6 — LRU = 2 (last used at step 4)
      |      |                |   → evict 2, load 4; last used: 3@6, 0@7, 4@8
  9   |  2   | [4, 3, 2]      | PF#7 — LRU = 3? No: 3@6, 0@7, 4@8 → LRU = 3
      |      |                |   Wait — 0 was evicted? No. After step 8: frames = {4,3,0}
      |      |                |   LRU among {4@8, 3@6, 0@7} = 3 (step 6) → evict 3, load 2
      |      |                |   last used: 0@7, 4@8, 2@9
  10  |  3   | [3, 4, 2]      | PF#8 — LRU = 0 (step 7) → evict 0, load 3; last used: 4@8, 2@9, 3@10
  11  |  0   | [3, 4, 0]      | PF#9 — LRU = 4 (step 8) → evict 4, load 0; last used: 2@9, 3@10, 0@11
  12  |  3   | [3, 4, 0]      | HIT — wait: frames = {3,4,0}? After step 11: {3,0,?}
      |      |                |   Correction: after step 11, frames = {3, 2(evicted?), 0}
      |      |                |   Let me restate cleanly from step 10 onward:
```

*Corrected clean walkthrough from beginning — tracking last-used step numbers:*

```css
Ref:  7   0   1   2   0   3   0   4   2   3   0   3   2   1   2   0   1   7   0   1
Step: 1   2   3   4   5   6   7   8   9   10  11  12  13  14  15  16  17  18  19  20

Step 1: page 7, empty frame → load 7. Frames: {7}. LRU tracking: {7:1}
Step 2: page 0, empty frame → load 0. Frames: {7,0}. Tracking: {7:1, 0:2}
Step 3: page 1, empty frame → load 1. Frames: {7,0,1}. Tracking: {7:1, 0:2, 1:3}
Step 4: page 2, miss, all full → LRU = 7 (last used step 1) → evict 7, load 2.
        Frames: {2,0,1}. Tracking: {0:2, 1:3, 2:4}                          [PF count: 4]
Step 5: page 0, HIT → update 0. Frames: {2,0,1}. Tracking: {1:3, 2:4, 0:5}
Step 6: page 3, miss → LRU = 1 (step 3) → evict 1, load 3.
        Frames: {2,0,3}. Tracking: {2:4, 0:5, 3:6}                          [PF count: 5]
Step 7: page 0, HIT → update 0. Frames: {2,0,3}. Tracking: {2:4, 3:6, 0:7}
Step 8: page 4, miss → LRU = 2 (step 4) → evict 2, load 4.
        Frames: {4,0,3}. Tracking: {3:6, 0:7, 4:8}                          [PF count: 6]
Step 9: page 2, miss → LRU = 3 (step 6) → evict 3, load 2.
        Frames: {4,0,2}. Tracking: {0:7, 4:8, 2:9}                          [PF count: 7]
Step 10: page 3, miss → LRU = 0 (step 7) → evict 0, load 3.
         Frames: {4,3,2}. Tracking: {4:8, 2:9, 3:10}                        [PF count: 8]
Step 11: page 0, miss → LRU = 4 (step 8) → evict 4, load 0.
         Frames: {0,3,2}. Tracking: {2:9, 3:10, 0:11}                       [PF count: 9]
Step 12: page 3, HIT → update 3. Tracking: {2:9, 0:11, 3:12}
Step 13: page 2, HIT → update 2. Tracking: {0:11, 3:12, 2:13}
Step 14: page 1, miss → LRU = 0 (step 11) → evict 0, load 1.
         Frames: {1,3,2}. Tracking: {3:12, 2:13, 1:14}                      [PF count: 10]
Step 15: page 2, HIT → update 2. Tracking: {3:12, 1:14, 2:15}
Step 16: page 0, miss → LRU = 3 (step 12) → evict 3, load 0.
         Frames: {1,0,2}. Tracking: {1:14, 2:15, 0:16}                      [PF count: 11]
Step 17: page 1, HIT → update 1. Tracking: {2:15, 0:16, 1:17}
Step 18: page 7, miss → LRU = 2 (step 15) → evict 2, load 7.
         Frames: {1,0,7}. Tracking: {0:16, 1:17, 7:18}                      [PF count: 12]
Step 19: page 0, HIT → update 0. Tracking: {1:17, 7:18, 0:19}
Step 20: page 1, HIT → update 1. Tracking: {7:18, 0:19, 1:20}
```

**Total page faults: 12**

### **Full Comparison**

| Algorithm | Page Faults (same reference string, 3 frames) | Practical? |
|-----------|-----------------------------------------------|------------|
| FIFO | 15 | Yes |
| Optimal (OPR) | 9 (minimum possible) | No — needs future knowledge |
| **LRU** | **12** | **Yes — best practical algorithm** |

*LRU is the best page replacement algorithm that can actually be implemented in a real OS.*

---

## **Implementing LRU**

### **Method 1: Counter (Time Field)**

**Mechanism:**
- Add a *time field* to each page table entry
- Maintain a single global *counter* (an integer starting at 0) that increments on every memory reference
- Every time a page is accessed, record the current counter value in that page's time field
- On a page fault (when eviction is needed): scan all page table entries, find the one with the **smallest time field value** — that is the LRU page → evict it

**Walkthrough (partial — same reference string):**

```css
Counter starts at 0, increments each step.

Step 1: access page 7 → load into frame. Page 7's time field = 1. Counter = 1.
        Page table time fields: {7:1}

Step 2: access page 0 → load into frame. Page 0's time field = 2. Counter = 2.
        Page table time fields: {7:1, 0:2}

Step 3: access page 1 → load into frame. Page 1's time field = 3. Counter = 3.
        Page table time fields: {7:1, 0:2, 1:3}

Step 4: access page 2 → PAGE FAULT (all frames full).
        Scan time fields: 7→1, 0→2, 1→3. Smallest = 7 (time field = 1) → evict 7.
        Load page 2, set time field = 4. Counter = 4.
        Page table time fields: {0:2, 1:3, 2:4}

Step 5: access page 0 → HIT. Update time field: 0→5. Counter = 5.
        Page table time fields: {1:3, 2:4, 0:5}

Step 6: access page 3 → PAGE FAULT.
        Scan: 1→3, 2→4, 0→5. Smallest = 1 (time=3) → evict page 1.
        Load page 3, time field = 6. Counter = 6.
        Page table time fields: {2:4, 0:5, 3:6}
        ... and so on
```

**Problem — Counter Overflow:**
- The counter is stored in a fixed-width integer (e.g., 32-bit or 64-bit)
- A 32-bit unsigned counter can hold values 0 to ~4.3 billion; in a heavily loaded system, it could eventually overflow
- A 64-bit counter is much safer in practice but the theoretical overflow issue remains

---

### **Method 2: Stack (Doubly Linked List)**

**Structure:**
- Maintain a doubly linked list treated as a *modified stack*
- **Top of stack = Most Recently Used (MRU) page**
- **Bottom of stack = Least Recently Used (LRU) page**
- The stack is *not* a pure LIFO stack — elements can be removed from anywhere (middle or bottom), not just the top

**Rules:**
1. When a page is **accessed** (whether a hit or a new load): remove it from its current position in the list and **push it to the top**
2. When **eviction is needed**: remove the element at the **bottom** — that is the LRU page

**Why doubly linked list?**
- Removing an element from the middle of the list (for a page hit on a page that is not at the top) requires updating both the previous and next pointers
- A doubly linked list supports O(1) removal from any position given a pointer to the node

**Walkthrough:**

```css
Step 1: load page 7.  Stack (top→bottom): [7]
Step 2: load page 0.  Stack: [0, 7]   (0 most recent)
Step 3: load page 1.  Stack: [1, 0, 7]  (1 most recent)

Step 4: page 2, miss — evict bottom (7 = LRU), load 2 to top.
        Stack: [2, 1, 0]

Step 5: page 0, HIT — move 0 from bottom to top.
        Stack: [0, 2, 1]

Step 6: page 3, miss — evict bottom (1 = LRU), load 3 to top.
        Stack: [3, 0, 2]

Step 7: page 0, HIT — move 0 (currently middle) to top.
        Stack: [0, 3, 2]

Step 8: page 4, miss — evict bottom (2 = LRU), load 4 to top.
        Stack: [4, 0, 3]

Step 9: page 2, miss — evict bottom (3 = LRU), load 2 to top.
        Stack: [2, 4, 0]

Step 10: page 3, miss — evict bottom (0 = LRU), load 3 to top.
         Stack: [3, 2, 4]

Step 11: page 0, miss — evict bottom (4 = LRU), load 0 to top.
         Stack: [0, 3, 2]

Step 12: page 3, HIT — move 3 (middle) to top.
         Stack: [3, 0, 2]

Step 13: page 2, HIT — move 2 (bottom) to top.
         Stack: [2, 3, 0]

Step 14: page 1, miss — evict bottom (0 = LRU), load 1 to top.
         Stack: [1, 2, 3]

Step 15: page 2, HIT — move 2 (middle) to top.
         Stack: [2, 1, 3]

Step 16: page 0, miss — evict bottom (3 = LRU), load 0 to top.
         Stack: [0, 2, 1]

Step 17: page 1, HIT — move 1 (bottom) to top.
         Stack: [1, 0, 2]

Step 18: page 7, miss — evict bottom (2 = LRU), load 7 to top.
         Stack: [7, 1, 0]

Step 19: page 0, HIT — move 0 (bottom) to top.
         Stack: [0, 7, 1]

Step 20: page 1, HIT — move 1 (bottom) to top.
         Stack: [1, 0, 7]
```

Total page faults: 12 (same result as counter method — both implement LRU correctly)

**LRU beyond OS page replacement:**
- The same LRU concept is used in **CPU cache design**, web proxy caches, database buffer pool management
- LRU is also a classic **DSA interview question** — "implement an LRU cache" using a doubly linked list + hash map — and frequently appears in interviews alongside OS theory questions

---

## **4. Counting-Based Algorithms**

*These are not commonly used in practice. Mentioned here as additional reference.*

Both algorithms maintain a **reference count** for each page: a counter that increments each time that page is accessed.

### **LFU — Least Frequently Used**
- **Logic:** An actively used page will have a high reference count; a rarely used page will have a low count
- **Rule:** On eviction, replace the page with the **smallest reference count** (least frequently accessed)
- **Problem:** A page heavily used early in execution accumulates a high count and is never evicted, even if it won't be needed again

### **MFU — Most Frequently Used**
- **Logic (opposite of LFU):** A page with a very small count was *just recently brought in* and has not had a chance to be used much yet — it is likely to be needed soon, so keep it
- **Rule:** On eviction, replace the page with the **highest reference count** (assumption: it has already served its purpose and is less likely to be needed urgently)

> *Neither LFU nor MFU is commonly used in real operating systems.* They are conceptual — their assumptions do not hold well across typical workloads.

---

## **Summary**

| Algorithm | Eviction rule | Page faults (example, 3 frames) | Practical? | Notes |
|-----------|--------------|----------------------------------|------------|-------|
| **FIFO** | Evict the page that has been in RAM the longest | 15 | Yes | Subject to Bélády's Anomaly |
| **Optimal (OPR)** | Evict the page whose next use is furthest in the future | 9 *(minimum possible)* | No | Needs future knowledge; used as benchmark only |
| **LRU** | Evict the page that was least recently used | 12 | Yes | Best practical algorithm; also used in cache design |
| **LFU** | Evict the page with the lowest access count | — | Rare | Counting-based |
| **MFU** | Evict the page with the highest access count | — | Rare | Counting-based; inverse of LFU |

**Key principle to remember:**
- *More page faults = more disk accesses = more overhead = slower system*
- The entire purpose of page replacement algorithms is to keep the page fault rate as low as possible
- LRU achieves this well in practice by exploiting temporal locality (recently used pages are likely to be used again soon)
