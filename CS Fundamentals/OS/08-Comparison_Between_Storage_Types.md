# Comparison Between Different Storage Types

---

## Storage Hierarchy in a Computer

```css
Closest to CPU
      │
      ▼
  Registers        ← inside CPU, fastest, smallest
      │
      ▼
    Cache           ← near CPU, very fast
      │
      ▼
  Main Memory       ← RAM, primary memory
      │
      ▼
Secondary Storage   ← Electronic Disk, Magnetic Disk, Optical Disk, Magnetic Tapes
      │
      ▼
Farthest from CPU
```

---

## Types of Storage

### Primary Memory

**1. Registers**
- *Smallest unit of storage* in the entire hierarchy.
- Located inside the CPU — this is where actual computation happens.
- Data and instructions are loaded into registers, and the CPU operates directly on them.
- Implemented using transistors and silicon — *physically closest to the CPU*.
- Stores 32 bits (in a 32-bit system) or 64 bits (in a 64-bit system) at a time.

**2. Cache**
- Additional memory layer between registers and main memory.
- Stores frequently used instructions and data so the CPU doesn't have to fetch from main memory repeatedly.
- *How it works:* The OS identifies instructions or data that a running program accesses repeatedly. It copies these into cache. Future accesses hit the cache instead of slower RAM.
- Cache access is faster than RAM access.
- Cache holds data in kilobytes (KB) typically.

**3. Main Memory (RAM)**
- Processes reside here during execution — a program is brought from disk into RAM to become a process.
- CPU communicates with RAM for its instructions and data.
- Capacity: typically in GBs today.

---

### Secondary Storage

- Includes:
  - **Electronic Disk** (e.g., SSD)
  - **Magnetic Disk** (e.g., HDD)
  - **Optical Disk** (e.g., CD/DVD)
  - **Magnetic Tapes**
- Stores programs, files, media, documents, projects — persistent data.
- Programs sit here until they are executed (at which point they are loaded into RAM).
- Capacity: in TBs.

---

## Comparison Table

| Property | Registers | Cache | Main Memory (RAM) | Secondary Storage |
|---|---|---|---|---|
| **Access Speed** | Fastest | Very Fast | Fast | Slowest |
| **Cost (per unit)** | Most expensive | Expensive | Moderate | Cheapest |
| **Storage Size** | Smallest (32/64 bits) | Small (KB) | Medium (GB) | Largest (TB) |
| **Volatility** | Volatile (data lost on power off) | Volatile | Volatile | Non-volatile (data persists) |
| **Distance from CPU** | Inside CPU | Very close | Close | Far |

---

## Why Are Registers and Cache Expensive?

- Built from high-quality materials (silicon, rare metals) designed for maximum speed.
- Require extremely precise manufacturing.
- The closer to the CPU, the higher the performance requirement — higher cost.
- *Making a 1 TB register would cost an enormous amount — no one would buy it. That's why sizes are kept small and cheaper storage handles the bulk.*

---

## Volatility Explained

- **Volatile:** Data is lost when power is removed. Registers, cache, and RAM are volatile.
- **Non-volatile:** Data persists after power-off. Secondary storage (HDD/SSD) is non-volatile.

*This is why you save your work to disk — RAM loses everything when the computer shuts down.*

---

## Summary

- **Registers:** Fastest, most expensive, smallest, inside CPU, volatile.
- **Cache:** Very fast, expensive, small (KB), near CPU, volatile — stores frequently accessed data.
- **RAM (Main Memory):** Fast, moderate cost, GBs, volatile — where processes live during execution.
- **Secondary Storage:** Slowest, cheapest, TBs, non-volatile — persistent file and program storage.
