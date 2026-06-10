# How the OS Creates a Process

---

## Program vs Process

**Program:**
- A compiled, executable file — ready to be run.
- Consists of machine-code instructions that the OS can interpret.
- *Resides on disk* (secondary storage).
- A program on disk cannot be executed by the CPU until it is brought into RAM.

**Process:**
- *Program under execution.*
- When the OS brings a program from disk into RAM, it becomes a process.
- A process is the unit of work done by a computer.

**Why processes?**
- The only way for a user to get the computer to perform a task is to write a program, compile it, and have the OS execute it as a process.

---

## How the OS Creates a Process — 5 Steps

### Step 1: Load the Program and Static Data into Memory

- The OS copies the program (compiled code) from disk into RAM.
- **Static data** is also loaded: pre-initialized variables declared in the program.
  - Example: `char* name = "lakshya";` — the string `"lakshya"` is static data that must be in memory before the process starts so the variable can be initialized correctly.
  - Another example: `int a = 0;` — the value `0` is static initialization data.

### Step 2: Allocate the Runtime Stack

- The OS allocates a **stack** region in memory for the process.
- The stack is used for:
  - Local variables of functions
  - Function arguments
  - Return values
  - Managing function call chains (recursion)
- When a function calls another function, the current function's state is pushed onto the stack; when the called function returns, the stack is "unwound."

### Step 3: Allocate the Heap

- The OS allocates a **heap** region in memory for the process.
- The heap is for **dynamic memory allocation** at runtime — memory requested via `malloc` / `new`.
- Unlike stack, heap memory must be managed explicitly:
  - Languages like Java have a **garbage collector** that automatically frees unused heap memory.
  - In C/C++, the developer must explicitly `free()` / `delete` allocated memory. Failing to do so causes memory accumulation.

### Step 4: Set Up I/O Infrastructure (I/O-Related Tasks)

- Three standard I/O handles (file descriptors) are created for the process:
  1. **Standard Input (stdin):** For reading input.
  2. **Standard Output (stdout):** For writing output.
  3. **Standard Error (stderr):** For writing error messages.
- These allow the process to perform I/O operations throughout its lifetime.
- In C++, `fprintf(stderr, "error message")` writes to the stderr handle.

### Step 5: Hand Off Control to `main()`

- The OS has completed setup. Now it starts the actual program execution.
- *The OS always starts execution at `main()`* — this is the entry point every OS knows to call.
- Once `main()` is called, the OS steps back — the process takes over and the CPU executes its instructions.

**What does `return 0` in `main()` mean?**
- The parent (OS/parent process) needs to know whether the child process completed successfully.
- `return 0` = **successful execution** → the OS interprets this as "completed without errors."
- `return -1` or `exit(-1)` = **unsuccessful execution** → the OS knows something went wrong; the program can handle this via error logic.

---

## Architecture of a Process in Memory

```css
High Address
┌─────────────┐
│   Stack     │  ← grows downward (local vars, function args, return values)
│      ↓      │
│             │
│      ↑      │
│    Heap     │  ← grows upward (dynamic memory via malloc/new)
├─────────────┤
│    Data     │  ← global variables, static data
├─────────────┤
│    Text     │  ← compiled code (instructions)
└─────────────┘
Low Address
```

- **Text segment:** The compiled machine code loaded from disk.
- **Data segment:** Global variables and statically initialized data.
- **Heap:** Dynamic memory; grows upward toward the stack.
- **Stack:** Function call management; grows downward toward the heap.

*At the start of execution, the OS places stack and heap far apart to give each room to grow.*

---

## Stack Overflow and Out-of-Memory Errors

**Stack Overflow:**
- Occurs when recursive calls have no base case — the stack keeps growing with each call.
- When the stack grows so large it meets the heap, the OS throws a **stack overflow** error.
- *Fix:* Always define a base case in recursive functions so the stack can unwind.

**Out-of-Memory Error (Heap Overflow):**
- Occurs when a program keeps allocating memory on the heap without freeing it.
- When the heap grows so large it meets the stack, the OS throws an **out-of-memory** error.
- *Fix:* Free / deallocate heap memory that is no longer needed (`free()` in C, `delete` in C++).

---

## Process Attributes and the PCB

The OS must manage many processes simultaneously. To do so, it maintains a **Process Table** — a data structure listing all active processes.

Each entry in the Process Table is a **PCB (Process Control Block)** — a data structure storing all information about one process.

*The OS identifies and manages a process entirely through its PCB.*

### PCB Fields

**1. Process ID (PID)**
- A unique integer identifier assigned to each process.
- The OS maintains a counter; each new process gets the next value.
- Used to distinguish between processes.

**2. Program Counter (PC)**
- *Tracks which instruction the process is currently executing.*
- Initialized to 0 (first instruction).
- After each instruction executes, PC increments to point to the next instruction.

**Fetch-Execute cycle:**
```css
while process is running:
  1. Fetch instruction at address stored in PC
  2. Increment PC (PC++)
  3. Execute the fetched instruction
  4. Repeat
```

*Without the program counter, the CPU would not know which instruction to execute next.*

**3. Process State**
- Records the current state of the process: New, Ready, Running, Waiting, Terminated.
- Covered in detail in the next lecture.

**4. Priority**
- An integer value indicating the scheduling priority of the process.
- Higher-priority processes get more CPU time.
- When spawning threads, developers can assign priority levels.

**5. Register Values (CPU Registers)**
- When a process is context-switched out (CPU taken away), the current state of all CPU registers is saved here.
- When the process is scheduled back in, these saved register values are restored into the actual CPU registers — allowing the process to resume exactly where it left off.
- *This is the core mechanism of context switching.*
- Registers saved include: Stack Pointer (SP), Base Pointer (BP), Control Registers (CR), and general-purpose registers.

**6. Open File List**
- List of all file descriptors (FDs) currently open by this process.
- Tells the OS which files the process is reading from or writing to.

**7. Open Device List**
- List of I/O device handles currently in use by this process.

---

## PCB Diagram

```css
┌───────────────────────────┐
│       Process ID          │
├───────────────────────────┤
│      Program Counter      │
├───────────────────────────┤
│       Process State       │
├───────────────────────────┤
│         Priority          │
├───────────────────────────┤
│    Register Values        │
│  (saved during ctx switch)│
├───────────────────────────┤
│     Open File List        │
├───────────────────────────┤
│     Open Device List      │
└───────────────────────────┘
```

*The OS accesses this PCB whenever it needs any information about a process — for scheduling, context switching, I/O management, or termination.*

---

## Process Table

```css
Process Table
┌────┬─────────────────────────────────────┐
│ P1 │  PCB of Process 1                   │
├────┼─────────────────────────────────────┤
│ P2 │  PCB of Process 2                   │
├────┼─────────────────────────────────────┤
│ P3 │  PCB of Process 3                   │
└────┴─────────────────────────────────────┘
```

- The OS maintains this table to know all currently running processes.
- *Each process has exactly one PCB; the PCB is the process from the OS's perspective.*
