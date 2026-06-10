## Starting with "Why" — Before the "What"

- **Application software** performs specific tasks for the user.
- **System software** operates and controls the computer system and provides a platform to run application software.
- The lecture deliberately starts with **why** an OS exists before defining what it is.
- A minimalistic computer has four core components: **CPU**, **GPU**, **Memory (RAM)**, **Disk**.
- Keep this in mind: imagine a world where **no OS exists at all**.

---

## Problem 1 — Resource Exploitation (No OS Scenario)

**Setup:** You open TikTok on a computer with no OS.

- The moment TikTok launches, it grabs **all available resources** — 100% CPU, 100% RAM, 100% GPU — because nothing is stopping it.
- Now you want to open PUBG alongside TikTok.
- PUBG cannot even launch. Before it can open, it hits a **hang state** — because TikTok has already hijacked every resource.
- Note: the machine actually **has enough resources** for both apps to run. The problem is not a resource shortage — the problem is that one app is taking everything with no management in place.

**The fix — insert an OS layer between apps and hardware:**

```css
WITHOUT OS:
  TikTok ──────────────────────→ Hardware (takes 100% of everything)
  PUBG   ──────────────────────→ Hang state (nothing left)

WITH OS:
  TikTok ──→ OS ──→ TikTok gets: 5% CPU, 10% RAM, 12% GPU
  PUBG   ──→ OS ──→ PUBG gets:  50% CPU, 50% RAM, 60% GPU
```

- Because TikTok is now limited to its allocated share, PUBG can load into memory.
- PUBG is a heavier app so it gets more — but both now run simultaneously.
- Resources are not going to just one app anymore; they are being distributed.
- This is called *Resource Management* — *<u>the first job of the OS.</u>*

**Technical note on the OS position in the system:**

- Once the OS layer is added, you can see that it sits *between* the apps/user and the hardware.
- Technically speaking, the OS is acting as an *Interface* here.

---

## Understanding "Interface" — The Bank Analogy

- Interface is a technical word — here is a plain-language example to make it concrete.

**Scenario:** You go to SBI (a bank) to withdraw ₹10,000.

- You walk in and talk to the cashier (uncle at the counter).
- You do **not** walk into the vault yourself to take your money.
- The cashier gives you a slip, you fill in your account number and amount, and the cashier goes into the vault and brings the cash to you.
- *The cashier is acting as the interface between you and the vault.*

```css
You (Customer)
      │
  [Cashier at counter] ← Interface
      │
  [Bank Vault / Cash]  ← The resource
```

**Mapping this analogy to the OS:**

| Bank world | OS world |
|---|---|
| You / Customer | User / App (e.g., TikTok) |
| Cashier at the counter | OS — the interface |
| Bank vault with cash | Hardware resources — CPU, GPU, RAM, etc. |

- The OS sits between the user/apps and the hardware, exactly like the cashier sits between you and the vault.
- Apps never directly touch the hardware. They always go through the OS.
- *The OS acts as an interface between the user/apps and the computer hardware.*

---

## Problem 2 — Bloated Apps and the DRY Principle Violation (No OS Scenario)

**Setup:** You are writing an app in C++ and need to allocate memory.

- You call `malloc(10)` — you ask for 10 bytes.
- You do **not** know how the OS finds free memory sectors internally, which RAM sectors it picks, or how it tracks them. You don't need to know. You just call `malloc` and get your memory.
- This is only possible **because the OS handles all of that** under the hood.

**Now remove the OS:**

- The TikTok developer would have to write their **own memory management code** from scratch — figuring out which RAM sectors are free, how to allocate, how to track, etc.
- The PUBG developer would **also** have to write the same memory management code from scratch.
- Both apps now contain the same hardware management code duplicated inside them.

**This violates the DRY Principle — "Do Not Repeat Yourself":**

- In coding: if you need an `isPrime()` function in 10 places, you write it **once** as a function and call it wherever needed. You never copy-paste the same logic 10 times.
- Here: both TikTok and PUBG are copy-pasting the same memory management logic. Every new app that runs on the machine has to do the same.
- Result: apps become **bloated**(*bulky*) — stuffed with infrastructure code that has nothing to do with their actual purpose.

```css
WITHOUT OS:
  TikTok = [TikTok app logic] + [memory management code] + [resource management code] + ...
  PUBG   = [PUBG app logic]   + [memory management code] + [resource management code] + ...
                                  ↑ same code duplicated everywhere — DRY violation, bloated apps

WITH OS:
  TikTok = [TikTok app logic only]
  PUBG   = [PUBG app logic only]
  OS     = [memory management + scheduling + process management + all hardware interaction code]
```

- With an OS, all resource management code lives **inside the OS's own codebase** — memory management, scheduling, process management — everything.
- Now a weather app developer only thinks about how the weather app works. They call `malloc` and memory appears. They do not think about how memory is found or tracked.
- *The OS prevents apps from becoming bloated by centralising all hardware interaction code inside itself.*
- This is the OS's **second job**: keeping apps lean by hiding the underlying hardware complexity.
- The technical term for hiding this complexity is **Abstraction**. Fancy word to use in interviews.

---

## Problem 3 — No Isolation or Protection (No OS Scenario)

**Setup:** Both TikTok and PUBG are loaded in RAM simultaneously. Each occupies some memory blocks.

- You can see their memory regions appear separate — but with no OS, there is **no enforcement** of that separation.

**What "isolation" actually means:**
- TikTok should not know that PUBG even exists in memory.
- TikTok should not be able to read or write to PUBG's memory addresses.
- Their memory regions must be **logically kept far apart** — not just physically placed at different addresses, but actively protected from each other.

**What happens without isolation — a dangerous example:**

- PUBG stores the player's health value at some memory address, say: `health = 100%`
- TikTok (with no restrictions) can scan through memory, find that address, and **overwrite it with 0%**.
- PUBG's health module reads the health value, sees 0%, and kills the player.
- You will be dead in the game — caused by a completely unrelated app writing to PUBG's memory.
- *This is what can happen without memory isolation — one app corrupting another app's live data.*

**With OS — how memory protection works:**

- The OS **logically tracks** exactly which memory range it has given to each app.
  - Example: TikTok → addresses 0 to location X
  - PUBG → addresses location X to location Y
- The OS enforces these boundaries. TikTok cannot access PUBG's range.
- TikTok does not even know PUBG is running — their **existences are kept separate**.
- Both apps have fully isolated memory. Neither can corrupt the other's data.

```css
RAM WITH OS:
  [0 ──────── X : TikTok's memory region]  [X ──────── Y : PUBG's memory region]
       OS enforces this boundary — TikTok cannot cross into PUBG's region
```

- *The OS provides Memory Protection and Isolation between any two or more apps running simultaneously.*
- This is the OS's **third job**.

---

## Functions of an OS — Summary

At this point, having seen all three problems, the functions of an OS can be listed clearly:

1. **Resource Management** *(also called Arbitration)*
   - Allocates CPU, RAM, GPU, and all other resources (memory, device, file, security, process, etc.) among apps.
   - No single app can monopolise everything.
   - Fancy interview word: *arbitration*.

2. **Interface Between Hardware and User/Apps**
   - Acts as intermediary — apps never directly access hardware.
   - All hardware access routes through the OS.
   - The OS sits between the two entities: computer hardware on one side, user/apps on the other.

3. **Hides Underlying Hardware Complexity** *(Abstraction)*
   - All hardware interaction code lives inside the OS's codebase — memory management, scheduling, process management.
   - App developers call `malloc()` without knowing or caring how memory is found internally.
   - The OS hides the complicated hardware side completely from the developer.
   - Fancy interview word: *abstraction*.

4. **Facilitates Execution of Application Programs**
   - Provides an environment where multiple programs run conveniently and efficiently.
   - No resource hijacking, no conflicts, no crashes from one app stealing another's resources.

5. **Isolation and Protection**
   - Each app's memory space is kept strictly separate and enforced.
   - One app cannot read or write another app's memory.

6. **Exclusive Access to Computer Hardware**
   - Hardware access belongs **only** to the OS. Users and apps do not have direct hardware access.
   - Any entity that needs hardware must go through the OS.

---

## Formal Definition of an Operating System

> **An Operating System is a piece of software that manages all the resources of a computer system — both hardware resources and software resources — and provides an environment in which a user can execute programs in a convenient and efficient manner by hiding the underlying complexity of the hardware and acting as a resource manager.**

- The OS provides the means for proper use of the resources in the operation of the computer system.
- An OS is made up of a **collection of system software**.

- The OS is itself a piece of software — just like the apps you write, except it manages everything underneath them.
- It manages both hardware resources (CPU, RAM, GPU, disk) and software resources.
- It provides a convenient and efficient environment — the user/developer doesn't have to worry about memory location, resource conflicts, or hardware details.
- Multiple apps can run simultaneously without hijacking each other.

**How does it achieve all this? Two key mechanisms:**

- By **hiding the underlying complexity of hardware** — abstraction
- By **acting as a resource manager** — arbitration

*Remember these two phrases for interviews.*
