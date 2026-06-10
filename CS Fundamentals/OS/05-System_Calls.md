# System Calls in Operating Systems

---

## Why System Calls Exist

- User Space and Kernel Space are **two separate, isolated spaces**.
- User Space applications do NOT have direct hardware access.
- Kernel Space has privileged hardware access.
- *System calls are the only mechanism by which user space can request kernel-space services.*

**Definition:**
> *A system call is a mechanism through which a user program can request a service from the kernel for which it does not have permission to perform directly.*

---

## How System Calls Work — The Flow

```css
User Application (User Mode)
        │
        │  calls libc (C library wrapper)
        ▼
   libc / wrapper
        │
        │  Software Interrupt → switches to Kernel Mode
        ▼
System Call Interface (SCI)
        │
        │  finds the kernel implementation mapped to this call
        ▼
   Kernel Implementation (written in C)
        │
        │  performs the actual hardware operation
        ▼
      Hardware
        │
        │  result returned → switch back to User Mode
        ▼
   User Application gets result
```

- The switch from User Mode to Kernel Mode happens via a **software interrupt**.
- *System calls are implemented in C* (low-level C that can directly interact with hardware).
- What you see as a command (e.g., `mkdir`) is an interface — internally, the kernel executes a C implementation that directly interacts with hardware.

---

## Example 1 — Creating a Directory

**Goal:** Create a folder called `movies`

**Two ways to trigger it:**
- GUI: Click "New Folder" → types `movies`
- CLI: `mkdir movies`

Both internally call the same system call path.

**Step-by-step flow:**
1. User types `mkdir movies` → *User Mode*
2. User space goes to the **System Call Interface (SCI)**
3. SCI looks up the kernel implementation corresponding to `mkdir`
4. CPU switches to **Kernel Mode** via software interrupt
5. Kernel's **File Management** component executes the `mkdir` implementation:
   - Finds the correct position in the directory tree
   - Adds a new node named `movies`
6. Directory created on disk
7. Success result returned → CPU switches back to **User Mode**
8. GUI shows the new `movies` folder

---

## Example 2 — Running a Process

1. User double-clicks `.exe` → *User Mode*
2. System call for execution is triggered via SCI
3. CPU switches to **Kernel Mode**
4. **Process Management** creates the process:
   - Allocates memory in RAM
   - Sets up stack, heap, I/O handles
5. Process created → result returned to User Space
6. CPU switches back to **User Mode**
7. Process appears in Task Manager / Activity Monitor

---

## User Mode ↔ Kernel Mode Switching

- Switching is done via **software interrupts**.
- A software interrupt says to the CPU: "Pause what you're doing and handle this important kernel-mode task."
- After the kernel task completes, control returns to user mode.

*This is the fundamental mechanism behind all OS interactions.*

---

## Categories of System Calls

There are **5 categories** of system calls:

### 1. Process Control
- Creating, terminating, waiting for, and setting attributes of processes
- Examples: `fork()` (creates child process), `exit()` (terminates process), `wait()` (waits for child)

**`fork()` explained:**
- Every process in a computer is a child of some other process — there is a root process (PID 0) from which all others descend.
- `fork()` creates a child process from a parent process.
- Example: When you run an executable from the terminal, the terminal is the parent; the new process is the child.

**`exit()` explained:**
- Deallocates memory, removes process from the process table, notifies the parent.

### 2. File Management
- `open`, `read`, `write`, `close`, `reposition`, `get/set attributes`, `chmod` (change mode), `chown` (change ownership)

**Permissions example (`chmod`):** A file owned by user `lakshya` can be set so only `lakshya` can read/write it — `ramesh` would be denied access.

**Reposition:** Moves the read/write pointer to a specific position in a file (e.g., start reading from the 20th character, not from the beginning).

### 3. Device Management
- `read`, `write`, `ioctl` (I/O control), set/get attributes, attach/detach devices

### 4. Information Maintenance
- `getpid`, `alarm`, `sleep`, `date`
- Used to get/set system time, date, process info, device info, etc.
- Example: Running `date` in the terminal triggers this type of system call.

### 5. Communication Management
- `pipe` (sets up message-passing between processes)
- `shmget` (gets shared memory segment for shared-memory IPC)
- `mmap` / `munmap` (maps/unmaps device or file into virtual memory)
- Used to implement IPC (Inter-Process Communication)

---

## Practical Demo Walkthrough

**Creating and running a shell script:**

1. Open terminal → create a file `hello.sh` with an infinite loop printing "Hello CodeHelp" every second.
2. **File Management system calls used:** `open` (creates file descriptor), `write` (writes content to file).
3. Run the script: `bash hello.sh`
4. **Process Control system call used:** `fork()` — terminal is the parent, `hello.sh` is the child process.
5. Check running processes: `ps -a`
6. **Information Management system call used:** `getpid` — retrieves the process ID displayed in the table.
7. Kill the process: `kill -9 <PID>`
8. **Process Control system call used:** `kill` — goes to kernel mode, finds the process, removes it from CPU and memory.

*The `man` command in Linux shows the manual for any system call. E.g., `man kill` shows what signals are available (2=interrupt, 3=quit, 9=kill, etc.).*

---

## Key Interview Answers

**Q: How do apps interact with the kernel?**
→ Through **system calls**.

**Q: How does User Mode switch to Kernel Mode?**
→ Via a **software interrupt**.

**Q: In what language are system calls implemented?**
→ **C** (low-level C with direct hardware access capability).

**Q: What is the System Call Interface (SCI)?**
→ An entry point into kernel space. It maps user-space system call requests to their corresponding kernel implementations.
