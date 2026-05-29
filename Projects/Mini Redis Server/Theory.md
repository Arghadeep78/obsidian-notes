# Mini Redis Server — Complete Theory & Project Guide

This document is written for someone who has some coding experience but is newer to systems programming concepts. It walks through every CS concept you need to understand this project, in the order you would encounter them if you built it yourself — starting from "what even is a server?" and ending with the exact trade-offs an interviewer will ask about.

---

## Table of Contents

- [[#1. Project Overview]]
- [[#2. Architecture]]
- [[#3. File Structure]]
- [[#4. Data Structures & Why]]
- [[#5. Concurrency Model]]
- [[#6. RESP Protocol]]
- [[#7. All Supported Commands]]
- [[#8. TTL / Expiration]]
- [[#9. Persistence]]
- [[#10. Server Lifecycle]]
- [[#11. Error Handling]]
- [[#12. Design Decisions & Trade-offs]]
- [[#13. Known Limitations]]
- [[#14. Build & Run]]
- [[#15. Testing]]
- [[#16. Interview Q&A]]

---

## 1. What Is Redis? What Are We Building?

### What is Redis?

Redis stands for **Remote Dictionary Server**. It is a program that runs on a machine and lets other programs store and retrieve data extremely fast — faster than a traditional database because it keeps everything **in memory** (RAM) rather than on a hard disk.

You connect to Redis over a network, send it a command like `SET name Alice`, and it stores that key-value pair. Later you can run `GET name` from anywhere on the network and get `Alice` back in under a millisecond.

Redis is used everywhere in production software:

- Caching database query results so pages load faster
- Storing user sessions so you stay logged in
- Rate limiting (counting how many requests a user has made)
- Message queues between services

### What are we building?

We are building a **mini version of Redis** from scratch in C++. It:

- Listens for TCP connections on port 6379 (the same port real Redis uses)
- Speaks the same wire protocol as Redis so you can connect to it with `redis-cli` — the official Redis command-line tool — exactly as you would a real Redis server
- Stores three types of data: **strings**, **lists**, and **hashes**
- Handles multiple clients at the same time using threads
- Saves data to disk so it survives restarts
- Supports 29 commands (28 unique + UNLINK as an alias for DEL)

This is a portfolio project that demonstrates: TCP networking, multi-threading, protocol parsing, data structure selection, and persistence — all in one compact codebase.

---

## 2. Networking Basics — How Computers Talk

Before writing a single line of server code, you need to understand how two programs on a network actually communicate.

### IP Addresses and Ports

Every machine on a network has an **IP address** — a unique identifier like `192.168.1.5`. But a machine runs many programs simultaneously (a browser, a music app, your server). The operating system uses **ports** to route incoming data to the right program.

Think of an IP address as a building's street address, and a port as the apartment number inside that building.

- Port 80: HTTP (web traffic)
- Port 443: HTTPS (secure web)
- Port 6379: Redis (what we use)
- Ports 1–1023: Reserved for system use; you need admin privileges
- Ports 1024–65535: Available for normal programs

When our server starts, it "claims" port 6379 by asking the operating system to route any incoming data on that port to our program.

### TCP vs UDP — Choosing a Protocol

There are two main ways to send data over a network:

**UDP (User Datagram Protocol):**

- Fire and forget — you send a packet, it may or may not arrive
- No guaranteed order
- Very fast, low overhead
- Used for: video streaming, DNS lookups, online games where speed matters more than reliability

**TCP (Transmission Control Protocol):**

- Reliable — the OS guarantees data arrives, in order, without corruption
- If a packet is lost, TCP automatically retransmits it
- Slower because of the overhead of guarantees
- Used for: web, email, Redis, databases — anything where you can't afford to lose data

We use TCP. When a client sends `SET foo bar`, we absolutely cannot afford to lose that command or receive it garbled.

### The TCP Handshake — How a Connection Is Established

Before any data flows, TCP does a three-step "handshake":

```
Client                        Server
  │                              │
  │──── SYN ────────────────────>│   "I want to connect"
  │                              │
  │<─── SYN-ACK ────────────────│   "OK, I'm ready"
  │                              │
  │──── ACK ────────────────────>│   "Connection established"
  │                              │
  │  (data can now flow both ways)
```

This is all handled automatically by the OS and the socket API — you just call `accept()` and a connected socket appears.

### What Is a Socket?

A socket is a **file descriptor** — an integer that represents an open I/O channel. In Unix/Linux, everything is a file: your actual files on disk, your keyboard, your screen, and network connections. They all get an integer ID (file descriptor) and you read/write them the same way.

When you create a socket, you get back an integer like `5`. You then:

- Call `bind()` to attach it to a port
- Call `listen()` to say "accept incoming connections here"
- Call `accept()` to wait for a client — this returns a _new_ socket specifically for that one client
- Call `recv()` to read data from the client socket
- Call `send()` to write data back
- Call `close()` when done

The server socket (the one you bind/listen on) stays open forever accepting new clients. Each client gets its own separate socket.

---

## 3. TCP Sockets — The Actual Code That Makes a Server

Here is the exact sequence of socket calls in [src/RedisServer.cpp](https://file+.vscode-resource.vscode-cdn.net/Users/arghadeep/Desktop/mini-redis-server/src/RedisServer.cpp), explained step by step.

### Step 1: Create the Socket

```cpp
server_socket = socket(AF_INET, SOCK_STREAM, 0);
```

- `AF_INET`: Use IPv4 addresses (as opposed to IPv6 which uses `AF_INET6`)
- `SOCK_STREAM`: Use TCP (stream = reliable, ordered). `SOCK_DGRAM` would be UDP.
- Returns an integer file descriptor, or -1 on error

### Step 2: Set Socket Options

```cpp
int opt = 1;
setsockopt(server_socket, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
```

This tells the OS: "if this port was recently in use by a dead process, let me reuse it immediately." Without this, after you kill and restart the server, you'd get "address already in use" for about 60 seconds while the OS waits in a state called `TIME_WAIT`. This is essential for development.

### Step 3: Bind to a Port

```cpp
sockaddr_in serverAddr{};
serverAddr.sin_family = AF_INET;
serverAddr.sin_port = htons(port);       // convert port number to network byte order
serverAddr.sin_addr.s_addr = INADDR_ANY; // listen on ALL network interfaces

bind(server_socket, (struct sockaddr*)&serverAddr, sizeof(serverAddr));
```

`htons()` — "host to network short" — converts a 16-bit number from your CPU's byte ordering (little-endian on most modern hardware) to the standard network byte order (big-endian). Numbers are bytes ordered differently on different machines; this ensures both sides agree.

`INADDR_ANY` means the server accepts connections from any IP address, not just one specific network interface. This is what you want for a server.

### Step 4: Listen

```cpp
listen(server_socket, 10);
```

Tells the OS to start queuing incoming connection requests. The `10` is the **backlog** — how many pending connections the OS will queue before refusing new ones. If 10 clients all connect simultaneously before you call `accept()`, the OS holds them. The 11th connection gets refused.

### Step 5: Accept Loop

```cpp
while (running) {
    int client_socket = accept(server_socket, nullptr, nullptr);
    // ... spawn a thread to handle client_socket
}
```

`accept()` **blocks** — your program pauses here (the thread is suspended by the OS, consuming no CPU) and waits until a client connects. When one does, the OS wakes the thread up and `accept()` returns a _new_ socket descriptor just for that client. The original `server_socket` stays open to accept the next client.

### Step 6: Receive and Send Data

```cpp
char buffer[1024];
int bytes = recv(client_socket, buffer, sizeof(buffer) - 1, 0);
// ... process the command in buffer ...
send(client_socket, response.c_str(), response.size(), 0);
```

`recv()` reads up to 1024 bytes from the client and returns how many bytes were actually read. A return value of 0 means the client disconnected cleanly. A negative value means an error occurred.

`send()` writes the response back. Returns -1 on error (e.g., client dropped the connection mid-send).

---

## 4. The RESP Protocol — How Redis Speaks

If you connect to our server with `redis-cli` and type `SET foo bar`, `redis-cli` doesn't actually send the plain text "SET foo bar". It speaks a structured protocol called **RESP** (Redis Serialization Protocol). Our server must parse this.

### Why a Protocol?

Imagine sending `SET foo bar` as plain text over TCP. How does the server know where the command ends? How does it handle values with spaces, like `SET greeting "hello world"`? A protocol defines a clear format with length headers so the receiver knows exactly how many bytes to read.

A complete, self-contained message in a protocol is called a **frame**. TCP is a stream — it delivers a continuous flow of bytes with no built-in message boundaries. The protocol's job is to define where one frame starts and the next begins, so the receiver can reconstruct complete commands even if they arrive in multiple TCP packets.

### RESP Format

Every RESP message uses **type prefixes** and **CRLF** (`\r\n`) as line terminators.

|Prefix|Meaning|Example|
|---|---|---|
|`+`|Simple string|`+OK\r\n`|
|`-`|Error|`-Error: key not found\r\n`|
|`:`|Integer|`:42\r\n`|
|`$`|Bulk string (arbitrary bytes)|`$5\r\nhello\r\n`|
|`*`|Array of the above|`*3\r\n...`|

### How a Command Is Sent

When you type `SET foo bar` in `redis-cli`, it sends this over TCP:

```
*3\r\n        <- array with 3 elements
$3\r\n        <- next element is 3 bytes long
SET\r\n       <- the 3 bytes
$3\r\n        <- next element is 3 bytes long
foo\r\n       <- the 3 bytes
$3\r\n        <- next element is 3 bytes long
bar\r\n       <- the 3 bytes
```

Each element in the array is a "bulk string": `$<length>\r\n<bytes>\r\n`. The length header means values can contain spaces, newlines, or binary data — no ambiguity.

### Parsing in Our Code

The function `parseRespCommand()` in [src/RedisCommandHandler.cpp](https://file+.vscode-resource.vscode-cdn.net/Users/arghadeep/Desktop/mini-redis-server/src/RedisCommandHandler.cpp) handles this:

```
1. Check if input starts with '*'
   - If not: it's plain text (e.g. from telnet) — split by whitespace
   - If yes: it's a RESP array

2. Read the number after '*' to know how many elements to expect

3. For each element:
   a. Read '$' to confirm it's a bulk string
   b. Read the length number
   c. Skip past '\r\n'
   d. Read exactly that many bytes as the token
   e. Skip past the trailing '\r\n'

4. Return the collected tokens as a vector<string>
```

**Validation:** The parser rejects element counts ≤ 0 or > 1024, catches `stoi()` exceptions for malformed headers, and stops early if the declared length extends past the actual input buffer.

### How Responses Are Encoded

Every command handler function returns a `std::string` that is already RESP-encoded:

```cpp
// Returning a simple success
return "+OK\r\n";

// Returning a string value
return "$" + std::to_string(value.size()) + "\r\n" + value + "\r\n";

// Returning nil (key not found)
return "$-1\r\n";

// Returning an integer
return ":" + std::to_string(count) + "\r\n";

// Returning an error
return "-Error: Key not found\r\n";

// Returning an array (e.g. KEYS command)
oss << "*" << keys.size() << "\r\n";
for (auto& k : keys)
    oss << "$" << k.size() << "\r\n" << k << "\r\n";
```

`redis-cli` then decodes these responses and presents them in human-readable form.

---

## 5. C++ Concepts Used in This Project

You don't need to be a C++ expert, but you need to understand these specific features.

### A Note on Big-O Notation

Throughout this guide you will see terms like O(1), O(N), and O(log N). These describe how an operation's time scales with the size of the data (N = number of elements):

- **O(1)** — constant time. Does the same amount of work regardless of how many elements there are. Accessing an array by index is O(1) — it doesn't matter if the array has 10 or 10 million elements, you jump straight to it.
- **O(N)** — linear time. Work grows proportionally with N. Shifting every element of a 1000-element array takes roughly 10× longer than shifting a 100-element array.
- **O(log N)** — logarithmic time. Slower than O(1) but far better than O(N). A binary search through 1,000,000 items takes about 20 steps (log₂ of 1,000,000 ≈ 20).
- **O(1) amortized** — the operation is sometimes slow (e.g., when a vector doubles its capacity) but on average over many calls it costs O(1). Like paying a big bill once every hundred operations — the average per-operation cost is still small.

### `std::string`

C++'s standard string type. Handles memory automatically — you don't need to worry about null terminators or buffer sizes like in raw C strings. Supports `.size()`, `+` for concatenation, `.find()`, `.substr()`, etc.

### `std::vector<T>`

A dynamically-sized array. It automatically resizes when you add elements. Key operations:

- `push_back(x)` — add to the end — O(1) amortized
- `insert(begin(), x)` — add to the front — O(N) because every existing element shifts right
- `pop_back()` — remove from end — O(1)
- `erase(begin())` — remove from front — O(N)
- `[i]` — random access by index — O(1)

### `std::unordered_map<K, V>`

A hash map. Stores key-value pairs where keys are hashed (put through a mathematical function that produces a bucket index) to locate their storage slot.

- `map[key] = value` — insert or update — O(1) average
- `map.find(key)` — lookup — O(1) average, returns an **iterator** (a pointer-like object that points to the found entry) or `map.end()` (a sentinel meaning "not found") if the key doesn't exist
- `map.erase(key)` — delete — O(1) average
- Order is not preserved — iterating gives keys in arbitrary order
- Worst case O(N) if all keys hash to the same bucket (very rare with good data)

Contrast with `std::map<K,V>`: a red-black tree. O(log N) for all operations, but keys are always sorted. We don't need sorted order, so `unordered_map` is the right choice.

### `std::mutex` and `std::lock_guard`

Before understanding `mutex`, you need to understand **threads** and **shared data**.

A **thread** is an independent path of execution inside your program. Your `main()` function is one thread. When you spawn a new thread, the OS gives it its own stack and runs it in parallel with your existing thread(s). Multiple threads inside the same process share the same memory — they see the same variables and the same data structures.

This sharing is powerful but dangerous. Imagine two threads both doing `counter++` at the same time. Under the hood, `counter++` is three steps: read the value, add 1, write it back. If two threads interleave these steps, both might read the same old value, both add 1, and both write back the same result — so you effectively only incremented once instead of twice. This is called a **data race** and produces wrong, unpredictable results.

A **mutex** (mutual exclusion) is a lock that solves this. Only one thread can "hold" (own) the mutex at a time. Any other thread that tries to acquire it will **block** (pause and wait) until the first thread releases it. This forces the three steps of `counter++` to always happen atomically — no other thread can sneak in between them.

```cpp
std::mutex my_mutex;

void safeWrite() {
    std::lock_guard<std::mutex> lock(my_mutex);  // Acquires lock here
    // ... do work ...
}  // lock_guard destructor runs here, releasing the lock automatically
```

`lock_guard` is a RAII wrapper — it acquires the lock in its constructor and releases it in its destructor. Even if the function throws an exception, the lock is guaranteed to be released. You never have to remember to call `unlock()`.

### `std::thread`

Creates a new OS thread that runs a function concurrently with your current thread. Think of it like spawning a new worker who runs a task alongside you — both of you work at the same time, independently.

```cpp
std::thread t([]() {
    // This code runs in a new thread
    doSomeWork();
});
t.detach();  // Let it run independently, don't wait for it
// OR:
t.join();    // Block here until the thread finishes
```

- **`detach()`** — you release ownership of the thread. It runs until it finishes on its own. You will never hear from it again; the OS cleans it up when it exits.
- **`join()`** — you wait for the thread to finish before your current code continues. Like saying "don't move on until that worker is done."

A **detached** thread runs completely independently. The program won't wait for it on exit. A **joined** thread is one the creating thread waits for.

### `std::atomic<bool>`

Suppose one thread reads a `bool` while another thread writes it at the exact same moment. Even for a simple `bool`, the CPU may cache the value in a register and never see the write from the other thread — or worse, the read and write may partially overlap, producing garbage. This is a data race even for a single variable.

`std::atomic<bool>` is a special type that the compiler and CPU treat as indivisible — reads and writes are guaranteed to be visible across threads immediately, with no possibility of tearing or caching issues. It is safe to read and write from multiple threads without a mutex.

`std::atomic<bool> running` ensures that when the signal handler sets `running = false`, the main thread's `while (running)` loop sees the updated value immediately, without any data race.

Without `atomic`, reading and writing `bool` from multiple threads is undefined behavior in C++.

### `std::chrono::steady_clock`

A clock for measuring time intervals. "Steady" means it only moves forward — it cannot be adjusted by the system administrator or NTP (Network Time Protocol). This is critical for TTL: if you used `system_clock` and the admin set the clock back one hour, all your expirations would suddenly be an hour further in the future. `steady_clock` is immune to that.

```cpp
auto expiry = std::chrono::steady_clock::now() + std::chrono::seconds(60);
// Later:
if (std::chrono::steady_clock::now() > expiry) { /* expired */ }
```

### RAII — Resource Acquisition Is Initialization

In C, you open a file and must remember to close it. You allocate memory and must remember to free it. You acquire a lock and must remember to release it. If your function returns early (or throws), you might forget — and you have a leak or a deadlock.

RAII is a C++ design pattern that ties resources to object **lifetime**. When an object is created (constructed), it acquires its resource. When the object goes out of scope — no matter how, even through an exception — its **destructor** runs automatically and releases the resource.

An **object going out of scope** means: the variable reaches the closing `}` of the block it was declared in, or the function returns. The destructor is a special method (`~ClassName()`) the compiler calls automatically at that moment.

`lock_guard`, `ifstream`, `ofstream`, and `std::thread` all use RAII. You don't forget to release resources because forgetting is impossible — the destructor always runs.

---

## 6. Data Structures — What We Store and Why

The heart of the project is `RedisDatabase` in [include/RedisDatabase.h](https://file+.vscode-resource.vscode-cdn.net/Users/arghadeep/Desktop/mini-redis-server/include/RedisDatabase.h) and [src/RedisDatabase.cpp](https://file+.vscode-resource.vscode-cdn.net/Users/arghadeep/Desktop/mini-redis-server/src/RedisDatabase.cpp). It holds four maps:

### `kv_store` — String Values

```cpp
std::unordered_map<std::string, std::string> kv_store;
```

Used by: `SET`, `GET`, `DEL`, `EXPIRE`, `TYPE`, `KEYS`, `RENAME`

A key maps directly to a single string value. `kv_store["name"] = "Alice"`. O(1) average for all operations. The simplest and most commonly used data type.

**Why `unordered_map` and not `map`?** We never need to iterate keys in sorted order. We only need lookup-by-exact-name. O(1) beats O(log N) for pure lookup workloads.

### `list_store` — Lists

```cpp
std::unordered_map<std::string, std::vector<std::string>> list_store;
```

Used by: `LPUSH`, `RPUSH`, `LPOP`, `RPOP`, `LLEN`, `LGET`, `LINDEX`, `LSET`, `LREM`

A key maps to an ordered sequence of strings. The outer `unordered_map` gives O(1) key lookup. The inner `vector<string>` holds the list elements in order.

**Performance breakdown for the inner vector:**

- `push_back(value)` — O(1) amortized — used by `RPUSH`
- `pop_back()` — O(1) — used by `RPOP`
- `insert(begin(), value)` — O(N) — used by `LPUSH` (shifts all elements right)
- `erase(begin())` — O(N) — used by `LPOP` (shifts all elements left)
- `vec[i]` — O(1) — used by `LINDEX`, `LSET`

**Key trade-off:** `LPUSH` and `LPOP` are O(N) because vector is a contiguous array and every element must physically shift. Real Redis uses a `quicklist` — a doubly-linked list where each node holds a small packed array of elements (rather than a single element), giving O(1) push/pop at both ends while keeping memory relatively compact. We could get O(1) at both ends with a much simpler fix: switch to `std::deque<string>`, which is a sequence container that efficiently supports insertion and deletion at both ends.

### `hash_store` — Hashes

```cpp
std::unordered_map<std::string, std::unordered_map<std::string, std::string>> hash_store;
```

Used by: `HSET`, `HGET`, `HMSET`, `HGETALL`, `HKEYS`, `HVALS`, `HLEN`, `HEXISTS`, `HDEL`

A key maps to a sub-map of field→value pairs. Think of it as a mini dictionary inside a dictionary. Example:

```
hash_store["user:1"] = {
    "name"  → "Alice",
    "email" → "alice@example.com",
    "age"   → "30"
}
```

Both the outer and inner lookups are O(1) average. This naturally models the Redis hash type.

### `expiry_map` — TTL Tracking

```cpp
std::unordered_map<std::string, std::chrono::steady_clock::time_point> expiry_map;
```

Used by: `EXPIRE`, `purgeExpired()`, `SET`, `DEL`, `RENAME`

Only keys that have an expiration appear here. It maps key name → the exact time_point when it should be deleted. Sparse: if you have 10,000 keys but only 50 have TTLs, this map has only 50 entries.

**Why a separate map?** If we stored TTL info directly in `kv_store`, we'd need a more complex value type everywhere. The separate map keeps things clean and memory-efficient.

### `db_mutex` — Thread Safety

```cpp
std::mutex db_mutex;
```

A single mutex that protects **all four maps** above. Every public method in `RedisDatabase` acquires this lock at the start and releases it at the end. This is called **coarse-grained locking** — one lock for everything. Only one thread can modify any of the four maps at a time.

The alternative is **fine-grained locking** — a separate lock per key or per store, so two threads working on different keys can proceed in parallel. The risk is **deadlock**: if Thread 1 holds Lock A and wants Lock B, while Thread 2 holds Lock B and wants Lock A, both threads wait forever — neither can proceed. With one global lock there is no nesting, so deadlock is impossible here.

---

## 7. Concurrency — Handling Many Clients at Once

### The Problem

The server needs to talk to many clients simultaneously. If Client A sends a big request and the server spends 50ms processing it, Client B shouldn't have to wait 50ms just to send `PING`.

A single-threaded server can only do one thing at a time. While it is blocked inside `recv()` waiting for bytes from Client A, Client B cannot even connect. We need **concurrency** — the ability to do multiple things at overlapping points in time.

### Our Solution: One Thread Per Client

```
Main thread:
  accept() → client_fd (for client #1)
  std::thread(handle client_fd).detach()   ← client #1 now runs independently
  
  accept() → client_fd (for client #2)
  std::thread(handle client_fd).detach()   ← client #2 now runs independently
  
  accept() → ...                           ← main thread always ready for new clients
```

Each accepted connection spawns a new OS thread. That thread loops:

```cpp
while (true) {
    int bytes = recv(client_socket, buffer, sizeof(buffer) - 1, 0);
    if (bytes <= 0) break;           // client disconnected
    string response = cmdHandler.processCommand(string(buffer, bytes));
    if (send(client_socket, response.c_str(), response.size(), 0) < 0) break;
}
close(client_socket);
// thread exits here
```

The thread lives until the client disconnects. Then the thread exits naturally and its memory is reclaimed.

### Why Detach?

We call `.detach()` immediately after creating the thread. This means:

- The main thread does NOT keep a reference to the client thread
- The main thread does NOT wait for client threads to finish
- Each client thread is fully self-managing

If we didn't detach (and didn't join), the `std::thread` destructor would call `std::terminate()` when the thread object goes out of scope — crashing the program. So we must either `join()` or `detach()`.

We can't `join()` because that would block the accept loop. So we detach.

### How the Shared Database Is Protected

All client threads call `RedisDatabase::getInstance()` and call methods on it. Multiple threads doing this simultaneously could corrupt the data.

For example: two threads both try to insert into `kv_store` at the same time. `unordered_map` internally resizes its bucket array when it gets full. If Thread 1 triggers a resize while Thread 2 is in the middle of reading, Thread 2 is reading memory that Thread 1 is simultaneously moving. The result is a crash or silent data corruption. This is a **data race** — two threads accessing the same memory at the same time, where at least one is writing, with no synchronization between them.

Every method in `RedisDatabase` starts with:

```cpp
std::lock_guard<std::mutex> lock(db_mutex);
```

This means:

- Thread 1 calls `set("foo", "bar")` — acquires `db_mutex`
- Thread 2 calls `get("foo")` — tries to acquire `db_mutex`, **blocks** (waits, doing nothing) because Thread 1 holds it
- Thread 1 finishes, `lock_guard` destructor releases `db_mutex`
- Thread 2 now acquires `db_mutex` and proceeds

Operations are **serialized** — they happen one at a time, in some order, but never overlapping. This eliminates all data races.

### `std::atomic<bool> running`

The main thread's accept loop is `while (running)`. The signal handler (which runs in a different context) sets `running = false` to stop the loop.

`std::atomic<bool>` ensures this write is immediately visible to the main thread without a mutex. Reading/writing a plain `bool` from multiple contexts is undefined behavior in C++.

### Thread Safety Summary

|Property|Status|
|---|---|
|No data races|Yes — all reads/writes protected by `db_mutex`|
|No deadlocks|Yes — only one lock exists, no nesting possible|
|Per-command atomicity|Yes — each command holds the lock for its entire duration|
|Multi-command transactions|No — `MULTI`/`EXEC` not implemented|
|Graceful thread drain on shutdown|No — `exit()` kills all detached threads immediately|

---

## 8. The Singleton Pattern — One Shared Database

### The Problem

We have one in-memory database. All client threads must access the _same_ database. How do we ensure there is always exactly one `RedisDatabase` instance, shared by everyone?

### The Singleton Pattern

```cpp
// In RedisDatabase.h
class RedisDatabase {
public:
    static RedisDatabase& getInstance();
private:
    RedisDatabase() = default;                           // can't construct from outside
    RedisDatabase(const RedisDatabase&) = delete;        // can't copy
    RedisDatabase& operator=(const RedisDatabase&) = delete; // can't assign
};

// In RedisDatabase.cpp
RedisDatabase& RedisDatabase::getInstance() {
    static RedisDatabase instance;   // created once, on the first call
    return instance;
}
```

**How it works:**

- `static RedisDatabase instance` inside a function: In C++, a `static` local variable is initialized the first time execution reaches that line, and only that one time. C++11 adds a guarantee: even if two threads race to call `getInstance()` simultaneously for the very first time, the compiler inserts an internal lock so only one of them runs the constructor. The other waits, then receives the already-constructed instance. You get exactly one object, created exactly once, safely.
- The constructor is `private` — nobody can do `RedisDatabase db;` anywhere else in the code.
- Copy and assignment are deleted — you can't accidentally make a second copy.

Every component in the project calls `RedisDatabase::getInstance()` and gets a reference to the exact same object.

**Trade-off:** Hard to unit test. You can't create two independent `RedisDatabase` objects for isolated tests. Every test shares the same global state.

---

## 9. TTL and Key Expiration

### What Is TTL?

TTL stands for **Time To Live**. In Redis, you can set a key to automatically delete itself after a certain number of seconds. Example:

```
SET session_token "abc123"
EXPIRE session_token 3600    // delete in 1 hour
```

This is used for caches, login sessions, rate limit counters, etc.

### How We Implement It

**Setting an expiration** (`EXPIRE key seconds`):

```cpp
// In RedisDatabase::expire()
expiry_map[key] = std::chrono::steady_clock::now() + std::chrono::seconds(seconds);
```

We compute the absolute time point at which the key should expire and store it. No timer is set — we just record _when_ the key should die.

**Checking expiration** (`purgeExpired()`):

```cpp
void RedisDatabase::purgeExpired() {
    // IMPORTANT: does NOT acquire db_mutex itself.
    // It assumes the calling method already holds the lock.
    auto now = std::chrono::steady_clock::now();
    for (auto it = expiry_map.begin(); it != expiry_map.end(); ) {
        if (now > it->second) {
            kv_store.erase(it->first);    // delete from string store
            list_store.erase(it->first);  // delete from list store
            hash_store.erase(it->first);  // delete from hash store
            it = expiry_map.erase(it);    // delete from TTL map, advance iterator
        } else {
            ++it;
        }
    }
}
```

`purgeExpired()` is a **private helper** — it does NOT acquire `db_mutex` itself. It assumes the calling method already holds the lock, which every caller does via `lock_guard`. This is a deliberate design: `std::mutex` is **non-reentrant** — if a thread that already holds the lock tries to acquire it again, it will block waiting for itself to release it, which can never happen. The thread hangs forever. This is called a **deadlock**. To avoid it, `purgeExpired()` trusts its callers to already hold the lock.

**Exact call sites** — `purgeExpired()` is called inside these methods (each of which already holds `db_mutex`):

|Method|Why it calls purgeExpired()|
|---|---|
|`get()`|Prevent returning an expired string value|
|`keys()`|Prevent listing expired keys|
|`type()`|Prevent reporting type of expired key|
|`del()`|Clean up before checking existence|
|`expire()`|Clean up before checking existence|
|`rename()`|Clean up before checking existence|

**Important limitation:** List and hash read operations (`lget`, `llen`, `lindex`, `hget`, `hgetall`, `hkeys`, `hvals`, `hlen`, `hexists`, `lpop`, `rpop`, etc.) do **NOT** call `purgeExpired()`. This means an expired list or hash key can still be read and returned until one of the above six methods is triggered on that key. This is a known bug — the correct fix is to call `purgeExpired()` at the start of every read method in `RedisDatabase`.

### Eviction Strategy: Lazy / On-Access

This is called **lazy eviction**: we only remove expired keys when someone tries to read them. Nobody "checks" keys proactively.

**Advantage:** Simple. No background thread needed. Zero CPU overhead when no reads happen.

**Disadvantage:** An expired key that nobody ever reads again stays in memory forever. Memory isn't reclaimed.

**What real Redis does:** Uses _both_ lazy eviction (on access) _and_ an **active eviction background process** that periodically samples random keys from the TTL map and removes expired ones. This bounds memory usage even for keys no one reads.

### Clearing TTL on SET

When you do `SET key value` on a key that already has a TTL, the old expiration is cleared:

```cpp
void RedisDatabase::set(const std::string& key, const std::string& value) {
    std::lock_guard<std::mutex> lock(db_mutex);
    kv_store[key] = value;
    expiry_map.erase(key);  // fresh SET resets any previous TTL
}
```

This matches real Redis behavior — `SET` always creates a fresh, permanent key.

### TTL Is Not Saved to Disk

The current persistence format does not store expiry times. On restart, `expiry_map` starts empty — every key that was about to expire now lives forever. This is a known limitation.

---

## 10. Persistence — Surviving Restarts

### The Problem

Redis is in-memory. If the server crashes or restarts, all data is lost — unless we save it to disk.

### Our Approach: Text File Snapshots

We save the entire database to a file called `dump.my_rdb` in a simple text format. This is conceptually similar to Redis's **RDB** (Redis Database) persistence mode, which takes periodic snapshots.

### File Format

Each line represents one record, prefixed with a type tag:

```
K <key> <value>
L <key> <item1> <item2> <item3>
H <key> <field1>:<val1> <field2>:<val2>
```

Example file:

```
K username alice
K session abc123
L queue task1 task2 task3
H user:1 name:Bob age:25 city:NYC
```

**K** = key-value string. **L** = list. **H** = hash (fields stored as `field:value` pairs separated by spaces).

### When Saves Happen

|Trigger|When|
|---|---|
|Startup|`main.cpp` calls `db.load("dump.my_rdb")` before any clients connect|
|Every 5 minutes|A background thread calls `db.dump("dump.my_rdb")` in an infinite loop|
|Ctrl+C (SIGINT)|`shutdown()` calls `db.dump()` before closing the socket|
|Accept loop exit|`run()` calls `db.dump()` as a final safety save|

### The dump() Function

```cpp
bool RedisDatabase::dump(const std::string& filename) {
    std::lock_guard<std::mutex> lock(db_mutex);   // freeze the DB during the write
    std::ofstream ofs(filename, std::ios::binary);
    if (!ofs) return false;

    for (const auto& kv : kv_store)
        ofs << "K " << kv.first << " " << kv.second << "\n";

    for (const auto& kv : list_store) {
        ofs << "L " << kv.first;
        for (const auto& item : kv.second)
            ofs << " " << item;
        ofs << "\n";
    }

    for (const auto& kv : hash_store) {
        ofs << "H " << kv.first;
        for (const auto& [f, v] : kv.second)
            ofs << " " << f << ":" << v;
        ofs << "\n";
    }
    return true;
}
```

**Note:** `db_mutex` is held for the entire duration of the dump. Because every client thread must acquire `db_mutex` before doing anything, this means every client is blocked — unable to read or write — until the entire file is written. Writing to disk is thousands of times slower than writing to RAM. For a large database with millions of keys, the dump could take several seconds, during which all clients appear to "freeze". For large databases this could cause a noticeable stall. Real Redis solves this by `fork()`-ing — creating a copy of the process — and letting the child process write the snapshot while the parent continues serving requests with no lock held.

### The load() Function

```cpp
bool RedisDatabase::load(const std::string& filename) {
    std::lock_guard<std::mutex> lock(db_mutex);
    std::ifstream ifs(filename, std::ios::binary);
    if (!ifs) return false;

    kv_store.clear(); list_store.clear(); hash_store.clear();  // start fresh

    std::string line;
    while (std::getline(ifs, line)) {
        std::istringstream iss(line);
        char type;
        iss >> type;
        if (type == 'K') { /* key value */ }
        else if (type == 'L') { /* key + space-separated items */ }
        else if (type == 'H') { /* key + field:value pairs split on ':' */ }
    }
    return true;
}
```

### Persistence Limitations

- **No atomic write:** If the process crashes mid-dump, the file is partially written and corrupt. Real fix: write to `dump.tmp`, then `rename("dump.tmp", "dump.my_rdb")`. Rename is atomic on Linux/macOS.
- **No checksums:** Corrupt lines are silently misread or skipped.
- **Space-delimited:** A key or value containing a space breaks the parser (e.g., `K greeting hello world` would parse `world` as a separate field).
- **No TTL persistence:** Expirations are lost on restart.

---

## 11. Signal Handling — Graceful Shutdown

### What Is a Signal?

A **signal** is an asynchronous notification the OS delivers to a running process. "Asynchronous" means it can arrive at any point in time — between any two instructions — regardless of what your code is currently doing. Your program does not poll for signals; the OS interrupts it and jumps to a special function called a **signal handler**.

Common signals:

- `SIGINT` — sent when you press `Ctrl+C`. Default action: terminate.
- `SIGTERM` — sent by `kill <pid>` or a process manager. Default action: terminate.
- `SIGSEGV` — sent when your code accesses invalid memory (segfault). Default action: crash + dump.

You can **register a custom handler** for most signals: a function that the OS calls instead of performing the default action. This is how we intercept `Ctrl+C` and save data before exiting.

### Why We Need This

If we let the process die immediately on `Ctrl+C`, any in-memory data not yet saved to disk is lost. We want to do a final save before exiting.

### How It Works

```cpp
// A C-style function pointer that the OS will call when SIGINT arrives
void signalHandler(int signum) {
    if (globalServer) {
        globalServer->shutdown();   // save DB, close socket
    }
    exit(signum);
}

// Register it at startup
void RedisServer::setupSignalHandler() {
    signal(SIGINT, signalHandler);
}
```

`globalServer` is a module-level pointer to the `RedisServer` instance. Signal handlers are C-level callbacks — they can't take arbitrary parameters, so we use a global pointer to reach the object we need.

`shutdown()` does three things:

1. Sets `running = false` — the accept loop checks this and exits
2. Calls `db.dump("dump.my_rdb")` — saves all data
3. Calls `close(server_socket)` — unblocks the `accept()` call so the loop can see `running == false`

---

## 12. Project File Structure and Responsibilities

```
mini-redis-server/
├── main.cpp                     Entry point: startup, background persist thread, run
├── include/
│   ├── RedisServer.h            TCP server class declaration
│   ├── RedisDatabase.h          In-memory data store (Singleton) declaration
│   └── RedisCommandHandler.h    RESP parser + command dispatcher declaration
├── src/
│   ├── RedisServer.cpp          Socket creation, accept loop, client threads, signal handling
│   ├── RedisDatabase.cpp        All four maps, mutex locking, TTL, dump/load
│   └── RedisCommandHandler.cpp  RESP parser + 29 command dispatch entries
├── test_all.sh                  Integration test script using redis-cli
├── Makefile                     Build instructions: g++ -std=c++17 -Wall -pthread -MMD -MP -O2
└── dump.my_rdb                  Auto-created on first dump; stores persisted data
```

### Separation of Concerns

Each file has a single, clearly bounded job:

|File|Owns|Does NOT touch|
|---|---|---|
|`main.cpp`|Startup/shutdown orchestration|No business logic|
|`RedisServer.cpp`|Network I/O — sockets, threads|No data storage, no protocol parsing|
|`RedisDatabase.cpp`|All data storage, TTL, persistence|No networking, no protocol|
|`RedisCommandHandler.cpp`|Protocol parsing, command dispatch|No networking (no sockets)|

This is **Separation of Concerns** — a fundamental software design principle. If you need to change how data is stored, you only touch `RedisDatabase.cpp`. If you add a new command, you only touch `RedisCommandHandler.cpp`.

---

## 13. Full Server Lifecycle — Start to Shutdown

### Startup Sequence (follows `main.cpp`)

```
1. Parse port from argv[1] (default 6379, range 1–65535)

2. RedisDatabase::getInstance().load("dump.my_rdb")
   └── Singleton created on first call (C++11 thread-safe static init)
   └── Restores kv_store, list_store, hash_store from file
   └── expiry_map starts empty (TTLs not persisted)

3. RedisServer server(port)
   └── Constructor: sets globalServer = this
   └── Calls setupSignalHandler(): registers SIGINT → signalHandler()
   └── server_socket = -1 (not opened yet)
   └── running = true

4. Background persist thread (detached):
   while (true) {
       sleep(300 seconds)
       db.dump("dump.my_rdb")
   }

5. server.run()  [blocks here until shutdown]
   └── socket() → create TCP socket
   └── setsockopt(SO_REUSEADDR)
   └── bind() → attach to port
   └── listen(backlog=10)
   └── Enter accept loop
```

### Accept Loop (inside `server.run()`)

```
while (running) {
    client_socket = accept(server_socket, ...)  // blocks here waiting for client
    
    if (client_socket < 0) {
        if (running) → print error
        else → break  // shutdown closed the socket, this is expected
    }
    
    std::thread([client_socket, &cmdHandler]() {
        char buffer[1024];
        while (true) {
            bytes = recv(client_socket, buffer, 1023, 0)
            if (bytes <= 0) break       // disconnect or error
            response = cmdHandler.processCommand(string(buffer, bytes))
            if (send(client_socket, ...) < 0) break
        }
        close(client_socket)
    }).detach();
}

// After loop exits:
db.dump("dump.my_rdb")  // final safety save
```

### Command Execution Path (for one command)

```
Client sends: "*3\r\n$3\r\nSET\r\n$3\r\nfoo\r\n$3\r\nbar\r\n"

recv() → string in buffer

processCommand(buffer)
  ├── parseRespCommand() → ["SET", "foo", "bar"]
  ├── cmd = "SET" (uppercased)
  ├── dispatch → handleSet(tokens, db)
  │     └── db.set("foo", "bar")
  │           ├── lock_guard acquires db_mutex
  │           ├── kv_store["foo"] = "bar"
  │           ├── expiry_map.erase("foo")
  │           └── lock released
  └── return "+OK\r\n"

send(client_socket, "+OK\r\n", 5, 0)

Client receives "+OK\r\n"
redis-cli displays: OK
```

### Shutdown Sequence (Ctrl+C)

```
1. OS delivers SIGINT to process
2. signalHandler() is called (registered at startup)
3. globalServer->shutdown():
   a. running = false              ← accept loop will stop on next iteration
   b. db.dump("dump.my_rdb")      ← final persistence save
   c. close(server_socket)         ← this unblocks the accept() call in the main thread
4. exit(SIGINT)                    ← process terminates
   └── All detached client threads are killed immediately (no graceful drain)
```

---

## 14. All 29 Commands — What Each Does

### General Commands

|Command|Syntax|What it does|Returns|
|---|---|---|---|
|`PING`|`PING`|Liveness check — always succeeds|`+PONG`|
|`ECHO`|`ECHO <msg>`|Echoes the message back|`+<msg>`|
|`FLUSHALL`|`FLUSHALL`|Deletes everything from all stores + expiry_map|`+OK`|

### String / Key-Value Commands

|Command|Syntax|What it does|Returns|
|---|---|---|---|
|`SET`|`SET key value`|Store a string value. Clears any existing TTL.|`+OK`|
|`GET`|`GET key`|Retrieve a string value. Calls purgeExpired() before reading (string ops only — list/hash reads do not).|`$<len>\r\n<value>` or `$-1` (nil)|
|`DEL`|`DEL key`|Delete a key from any store + clear TTL|`:1` (deleted) or `:0` (not found)|
|`UNLINK`|`UNLINK key`|Alias for DEL (same implementation)|`:1` or `:0`|
|`KEYS`|`KEYS`|Return all keys across all three stores|`*N\r\n...` array|
|`TYPE`|`TYPE key`|Report which type the key is|`+string`, `+list`, `+hash`, or `+none`|
|`EXPIRE`|`EXPIRE key seconds`|Set a TTL on a key|`+OK` or error if key not found|
|`RENAME`|`RENAME old new`|Move key + value + TTL to a new name|`+OK` or error|

### List Commands

Lists are ordered sequences. Index 0 is the leftmost (head) element.

|Command|Syntax|What it does|Performance|
|---|---|---|---|
|`LPUSH`|`LPUSH key v1 [v2...]`|Prepend to front. Each value pushed individually left-to-right.|O(N) per element (vector insert at position 0)|
|`RPUSH`|`RPUSH key v1 [v2...]`|Append to back|O(1) per element (push_back)|
|`LPOP`|`LPOP key`|Remove and return the first element|O(N) (vector erase at position 0 shifts everything)|
|`RPOP`|`RPOP key`|Remove and return the last element|O(1) (pop_back)|
|`LLEN`|`LLEN key`|Count of elements|O(1)|
|`LGET`|`LGET key`|Return entire list as an array|O(N)|
|`LINDEX`|`LINDEX key index`|Element at index (negative = from end, -1 = last)|O(1)|
|`LSET`|`LSET key index value`|Overwrite element at index|O(1)|
|`LREM`|`LREM key count value`|Remove matching elements. count>0: from head; count<0: from tail; count=0: all|O(N)|

### Hash Commands

Hashes store field→value pairs inside a key. Like a dictionary within a key.

|Command|Syntax|What it does|
|---|---|---|
|`HSET`|`HSET key field value`|Set one field. Returns 1 if new field, 0 if updated.|
|`HGET`|`HGET key field`|Get one field's value|
|`HMSET`|`HMSET key f1 v1 [f2 v2...]`|Set multiple fields at once (must come in field/value pairs)|
|`HGETALL`|`HGETALL key`|Return all fields and values as a flat array: [f1, v1, f2, v2, ...]|
|`HKEYS`|`HKEYS key`|Return all field names|
|`HVALS`|`HVALS key`|Return all field values|
|`HLEN`|`HLEN key`|Count of fields in the hash|
|`HEXISTS`|`HEXISTS key field`|1 if field exists, 0 if not|
|`HDEL`|`HDEL key field`|Remove one field; 1 if it existed|

---

## 15. Error Handling — All Five Layers

The server has five distinct layers of defense against bad input.

### Layer 1 — RESP Parse Errors

In `parseRespCommand()`: handles malformed or truncated RESP frames.

```cpp
// Bad element count
try { numElements = std::stoi(...); }
catch (...) { return {}; }   // empty vector = "parsing failed"

if (numElements <= 0 || numElements > 1024) return {};

// Frame truncated mid-element
if (pos + len > input.size()) break;
```

The caller receives an empty `vector<string>` and returns `-Error: Empty command\r\n`.

### Layer 2 — Argument Count Validation

Every handler checks it has enough arguments before doing anything:

```cpp
static std::string handleSet(const vector<string>& tokens, RedisDatabase& db) {
    if (tokens.size() < 3)
        return "-Error: SET requires key and value\r\n";
    // ...
}
```

### Layer 3 — Numeric Argument Parsing

Commands that accept integers (EXPIRE, LREM, LINDEX, LSET) wrap `stoi()` in try-catch:

```cpp
try {
    int seconds = std::stoi(tokens[2]);
    // use seconds...
} catch (const std::exception&) {
    return "-Error: Invalid expiration time\r\n";
}
```

### Layer 4 — Socket I/O Errors

The client thread checks every `recv()` and `send()` return value:

```cpp
int bytes = recv(client_socket, buffer, sizeof(buffer) - 1, 0);
if (bytes <= 0) break;      // 0 = clean disconnect, <0 = error — exit thread

if (send(client_socket, ...) < 0) break;   // send failed — exit thread
```

When the loop breaks, `close(client_socket)` is called and the thread exits. The client's connection is cleaned up regardless of how it ended.

### Layer 5 — File I/O

```cpp
std::ofstream ofs(filename);
if (!ofs) return false;   // caller logs the failure
```

`dump()` and `load()` return `bool` to signal success/failure. Callers print an error message but the server continues running.

---

## 16. Design Decisions and Trade-offs

These are the decisions most likely to come up in an SDE-1 interview. Know the "why" and the trade-off for each.

### 1. Singleton for RedisDatabase

**Decision:** Use a Singleton via a static local variable in `getInstance()`.

**Why:** Guarantees exactly one database shared across all threads. C++11 makes static local initialization thread-safe. Any component can call `getInstance()` without needing a reference passed to it.

**Trade-off:** Hard to test in isolation. You can't create two separate DB instances for isolated unit tests. All tests share global state and must clean up after themselves.

**What an interviewer wants to hear:** "The Singleton is convenient here because of the thread-safe static init guarantee in C++11, but it creates tight coupling and makes testing harder. For a production system I'd use dependency injection — pass a `Database*` to components that need it."

### 2. Coarse-Grained Mutex (Single Global Lock)

**Decision:** One `std::mutex db_mutex` protects all three stores.

**Why:** Dead simple. No lock ordering to think about. Zero risk of deadlock. Easy to reason about correctness.

**Trade-off:** All concurrent clients serialize on this one lock. If Client A is doing a slow `LREM` on a 100,000-element list, every other client waits.

**Scaling fix:** Use `std::shared_mutex` — a **reader-writer lock**. A regular mutex forces everyone to take turns, even two threads that only want to _read_ (reading simultaneously is always safe — the danger is only when someone writes). A reader-writer lock allows any number of threads to read at the same time. When a writer arrives, it waits for all current readers to finish, then gets exclusive access. For a read-heavy workload (more GETs than SETs) this is a big win. Further improvement: **shard** the mutex — use a separate lock per key or per store type, so two threads working on different keys can run in parallel.

**What real Redis does:** Single-threaded event loop — no mutex at all because only one thread ever touches the data. **I/O multiplexing** (via a Linux kernel feature called `epoll`) lets one thread watch thousands of sockets simultaneously and wake up only when one has data ready. The thread processes that client, then immediately loops back and checks the others. Because there is only one thread, there is no shared-data problem — no mutex needed. The database is never touched from two threads at once.

### 3. One Thread Per Client

**Decision:** `std::thread([client_socket](){ ... }).detach()` for every accepted connection.

**Why:** Simple mental model. Each client has its own stack and its own blocking recv/send loop.

**Trade-off:** Each OS thread uses ~8MB of **stack memory** (the memory reserved for that thread's local variables and function call frames) and takes time to create and destroy. At 10,000 simultaneous clients, you have 10,000 threads — the OS **scheduler** (the part of the OS that decides which thread runs on which CPU core at any moment) struggles to switch between them efficiently, and you use ~80GB of stack space.

**Scaling fix:** A **thread pool** — create a fixed set of worker threads (e.g., one per CPU core) at startup and never create more. When a client has work to do, push it onto a shared **queue** (a list where items are added at one end and taken from the other, first-in-first-out); an idle worker picks it up. Combined with **non-blocking sockets** and `epoll` for I/O multiplexing — one thread can efficiently monitor thousands of sockets.

> **Non-blocking socket:** By default, `recv()` blocks — if no data has arrived yet, the calling thread sleeps until it does. With a non-blocking socket, `recv()` returns immediately with a special error code (`EAGAIN`) meaning "nothing ready yet, try later." This lets a single thread call `recv()` on many sockets in a loop without ever getting stuck waiting for one slow client. Combined with `epoll` (which tells you _which_ sockets actually have data), the thread only calls `recv()` on sockets it already knows have bytes waiting.

### 4. Lazy TTL Eviction

**Decision:** Expired keys are only removed when accessed — on-read eviction only.

**Why:** No background thread needed. Simple. Zero CPU overhead during idle periods.

**Trade-off:** An expired key that nobody reads again stays in memory forever. Memory grows without bound under high TTL usage.

**Fix:** Add a background thread that randomly samples keys from `expiry_map` every 100ms and removes expired ones — this is what real Redis does.

### 5. `std::vector` for List Elements

**Decision:** `vector<string>` instead of `deque<string>` or a linked list.

**Why:** Vector is the simplest STL container. It stores all elements in **contiguous memory** — one unbroken block of RAM. CPUs read memory in chunks called **cache lines** (typically 64 bytes). Because vector elements sit next to each other, when you access one element the CPU pre-loads the neighboring elements into its fast on-chip cache for free. This is called **cache-friendly** access. A linked list, by contrast, scatters nodes across RAM — each access is a new cache miss. Vector therefore iterates much faster in practice.

**Trade-off:** `LPUSH` (prepend) and `LPOP` (remove first) are O(N) because all elements must physically shift one slot right (or left) in memory. For long lists this is a performance cliff.

**Fix:** Change `vector<string>` to `std::deque<string>` — O(1) at both ends, O(1) random access, minimal code change. Real Redis uses a `quicklist` (a doubly-linked list where each node holds a small, tightly-packed array of elements rather than a single element) — this keeps memory relatively cache-friendly while still allowing O(1) push/pop at both ends.

### 6. Text-Based Persistence Format

**Decision:** Human-readable space-delimited text file.

**Why:** Easy to implement and debug. You can open `dump.my_rdb` in a text editor and read it directly.

**Trade-off:** Breaks if keys or values contain spaces. No checksums — silent corruption. Slow to parse. No TTL serialization. Large file size for big datasets.

**Fix:** Binary format (like real Redis RDB), or JSON/MessagePack with proper escaping and checksums.

### 7. Fixed 1024-byte Receive Buffer

**Decision:** `char buffer[1024]` per `recv()` call, no buffering or reassembly.

**Why:** Keeps networking code minimal.

**Trade-off:** Any RESP frame longer than 1024 bytes is silently truncated and will be misparsesd. A `SET key "large_value..."` with a value > ~990 bytes would fail silently.

**Fix:** Use a dynamic buffer. Accumulate bytes from `recv()` calls, detect complete RESP frames by parsing headers, and only process when a full frame is available.

### 8. Case-Insensitive Commands

**Decision:** `std::transform(cmd, ..., ::toupper)` before dispatching.

**Why:** Redis is case-insensitive — `SET`, `set`, `Set` all work. Doing it once at dispatch time means every handler only needs to handle one case.

---

## 17. Known Limitations and How to Fix Them

|Limitation|Impact|Fix|
|---|---|---|
|`LPUSH`/`LPOP` are O(N)|Slow for large lists|Use `std::deque<string>`|
|Single coarse mutex|All clients serialize; bottleneck under load|`shared_mutex` for readers, or per-shard locks|
|1024-byte recv buffer|Silently truncates large commands|Dynamic buffer with frame reassembly|
|No TTL persistence|Expirations lost on restart|Store expiry timestamps in dump file|
|`load()` does not clear `expiry_map`|On reload, stale TTL entries for keys that were deleted could theoretically linger|Clear `expiry_map` at the start of `load()`, same as the three data stores|
|`purgeExpired()` not called in list/hash reads|An expired list or hash key can still be returned by LGET, HGET, etc.|Call `purgeExpired()` at the start of every read method|
|Space-delimited dump format|Breaks with spaces in keys/values|Binary format or quoted strings|
|No atomic writes to disk|Crash mid-dump corrupts file|Write to temp file, then rename atomically|
|No `MULTI`/`EXEC`|No transactions|Per-client transaction state, command queue|
|Detached threads on shutdown|In-flight requests killed by `exit()`|Thread pool with join on shutdown|
|No AUTH|No access control|Password challenge on connect|
|No SELECT|Single keyspace|Multiple databases via array of maps|
|One thread per client|Doesn't scale past ~1000 clients|`epoll` + thread pool|
|Lazy-only TTL eviction|Expired-but-unread keys stay in memory|Background reaper thread|

---

## 18. Build and Run

### Prerequisites

- `g++` with C++17 support (GCC 7+ or Clang 5+)
- `make`
- `redis-cli` (for testing; install via `brew install redis` on macOS)

### Build

```bash
cd mini-redis-server
make
# Produces: ./my_redis_server
```

The Makefile compiles with: `g++ -std=c++17 -Wall -pthread -MMD -MP -O2`

|Flag|What it does|
|---|---|
|`-std=c++17`|Enable C++17 features (`std::optional`, structured bindings, etc.)|
|`-Wall`|Enable all common compiler warnings — catches bugs at compile time|
|`-pthread`|Link the pthreads library, the OS-level implementation of `std::thread`|
|`-MMD -MP`|Auto-generate `.d` dependency files so `make` knows which `.cpp` files to recompile when a header changes|
|`-O2`|Optimization level 2 — faster binary without debug-hostile transformations|

### Run

```bash
./my_redis_server          # Port 6379 (Redis default)
./my_redis_server 6380     # Custom port
```

On startup you will see:

```
Database Loaded From dump.my_rdb      (or: No dump found... starting empty)
Redis Server Listening On Port 6379
```

### Connect and Test

```bash
redis-cli -p 6379
> PING
PONG
> SET name Alice
OK
> GET name
"Alice"
> EXPIRE name 10
OK
> KEYS
1) "name"
```

### Run Integration Tests

```bash
./test_all.sh   # requires server already running on port 6379
```

The test script pipes all 29 commands through `redis-cli` and checks expected outputs.

### Shutdown

Press `Ctrl+C`. The server will print:

```
Database Dumped to dump.my_rdb
Server Shutdown Complete!
```

---

## 19. Interview Q&A — Prepared Answers

These are questions a FANG SDE-1 interviewer is likely to ask. The answers are written at the level of an SDE-1 — technically precise, not overly academic.

---

**Q: Walk me through what happens when a client runs `SET foo bar`.**

> `redis-cli` encodes the command as a RESP array: `*3\r\n$3\r\nSET\r\n$3\r\nfoo\r\n$3\r\nbar\r\n` and sends it over TCP. The server's accept loop already spawned a dedicated thread for this client when it connected. That thread's `recv()` returns the bytes into a 1024-byte buffer.
> 
> `processCommand()` is called. `parseRespCommand()` detects the `*` prefix and parses out three tokens: `["SET", "foo", "bar"]`. The command is uppercased to `"SET"` and dispatched to `handleSet()`.
> 
> `handleSet()` calls `db.set("foo", "bar")`. Inside `set()`, a `lock_guard` acquires `db_mutex`, stores `kv_store["foo"] = "bar"`, and erases any existing entry in `expiry_map["foo"]` (clearing old TTL). The lock is released automatically.
> 
> `handleSet()` returns `"+OK\r\n"`. The client thread calls `send()` with this response. `redis-cli` receives `+OK` and displays `OK`.

---

**Q: How does your server handle multiple clients at the same time?**

> Each accepted connection spawns a detached `std::thread`. These threads all share the `RedisDatabase` singleton. Access to the database is protected by a single `std::mutex` — each method acquires a `lock_guard` so operations don't interleave. Clients run concurrently on separate threads, but database operations are serialized through the mutex. So we get true parallelism at the I/O level (multiple clients can be in `recv()` simultaneously) but sequential access to the shared data store.

---

**Q: What's the biggest performance bottleneck and how would you fix it?**

> The single global `db_mutex` is the primary bottleneck — all concurrent clients serialize on every database operation. Two fixes depending on target scale:
> 
> Short-term: Replace `std::mutex` with `std::shared_mutex`. Read commands (`GET`, `HGET`, etc.) acquire a shared (reader) lock, allowing multiple concurrent reads. Write commands (`SET`, `HSET`, etc.) acquire an exclusive (writer) lock. For read-heavy workloads this is a significant improvement.
> 
> Long-term: Move to an event-driven architecture like real Redis — a single-threaded event loop with `epoll`, non-blocking sockets, and an internal command queue. This eliminates mutex contention entirely since the database is only ever touched from one thread. I/O multiplexing handles thousands of connections in that one thread.

---

**Q: How does your TTL implementation work? What are its weaknesses?**

> `EXPIRE key 60` stores `steady_clock::now() + 60s` in `expiry_map`. Before certain operations, `purgeExpired()` scans `expiry_map` and deletes any entries past their expiry from all stores.
> 
> Weaknesses: (1) Lazy eviction only — expired keys that are never read again stay in memory forever. Memory grows without bound. (2) `purgeExpired()` is only called before string-type operations and key-metadata operations (`GET`, `KEYS`, `TYPE`, `DEL`, `EXPIRE`, `RENAME`) — list and hash read commands (`LGET`, `HGET`, etc.) do NOT call it, so an expired list or hash key can still be read. This is a bug. (3) TTLs are not saved to disk, so they're lost on restart. (4) `purgeExpired()` also does NOT acquire `db_mutex` itself — it assumes the caller already holds it, which all callers do.
> 
> Fix: call `purgeExpired()` at the start of every read method in `RedisDatabase`. Add a background reaper thread. Store TTL timestamps in the dump file.
> 
> Fix: Add a background thread that samples random keys from `expiry_map` every 100ms and removes expired ones. Also store TTL timestamps in the dump file.

---

**Q: Why did you use `unordered_map` instead of `map`?**

> `unordered_map` gives O(1) average-case lookup, insert, and delete via hashing. `map` gives O(log N) via a balanced BST. For a key-value cache, access patterns are lookup-by-exact-name — not range scans or sorted iteration. O(1) is clearly better. The downside is that `unordered_map` returns keys in arbitrary order — but Redis's `KEYS` command makes no ordering guarantees anyway, so that's fine.

---

**Q: How does your persistence work? Is it crash-safe?**

> Data is dumped to `dump.my_rdb` in a human-readable text format: a background thread does it every 5 minutes, and the SIGINT handler does it on shutdown. On startup, the file is loaded to restore state.
> 
> It is not crash-safe. If the process crashes mid-dump, the file is partially written and the next load will either fail or restore corrupted state. The correct approach is: write to a temporary file (`dump.tmp`), `fsync()` it to ensure it's on disk, then `rename("dump.tmp", "dump.my_rdb")`. On Linux and macOS, `rename()` is atomic — a reader either sees the old file or the new file, never a half-written one. This is what Redis's BGSAVE does.

---

**Q: What design pattern does `RedisDatabase` use and why?**

> It's a Singleton using the Meyer's Singleton pattern — a `static` local variable in `getInstance()`. C++11 guarantees this initialization is thread-safe, so even if ten threads call `getInstance()` simultaneously on first use, only one instance is created.
> 
> The rationale: there should be exactly one in-memory database, shared across all connection-handling threads. Rather than passing a `RedisDatabase*` to every component, the Singleton gives any code a clean access point. The downside is it makes unit testing harder — you can't create an isolated DB instance per test — and it creates implicit coupling between components.

---

**Q: How would you scale this to handle 100,000 concurrent connections?**

> The current one-thread-per-client model breaks down around 1,000–10,000 threads because of OS scheduler overhead and memory (each thread typically uses 8MB of stack). I'd redesign in two parts:
> 
> Networking layer: Use `epoll` (Linux) or `kqueue` (macOS) for I/O multiplexing. These are OS kernel features that let one thread say "tell me when any of these 100,000 sockets has data ready" and then sleep until something actually needs attention. A single thread can monitor thousands of sockets and process whichever ones are ready. Combine with a **thread pool** (a fixed number of workers, one per CPU core) that pulls ready events from a queue. Sockets must be **non-blocking** — when there is no data, `recv()` returns immediately with an error code instead of sleeping — so no worker thread ever stalls waiting for a slow client.
> 
> Database layer: Replace the single global mutex with **sharded locks** — divide the keyspace into N buckets by hashing each key (e.g., `hash(key) % 64`), and give each bucket its own lock. Two operations on keys that hash to different buckets can now proceed in parallel. For even more scale, move to **lock-free data structures** that use CPU **atomic operations** (hardware-level instructions that read-modify-write a memory location in a single indivisible step, with no mutex at all).

---

**Q: What would you add if you had two more weeks?**

> In priority order:
> 
> 1. **Safe persistence** — write to a temp file, then atomic rename. Add TTL timestamps to the dump format.
> 2. **Dynamic receive buffer** — frame reassembly so commands larger than 1024 bytes work correctly.
> 3. **`std::deque` for lists** — makes `LPUSH`/`LPOP` O(1) with a one-line change.
> 4. **TTL background reaper thread** — samples random keys every 100ms, reclaims memory from expired-but-unread keys.
> 5. **`MULTI`/`EXEC` transactions** — per-client command queue, execute atomically.
> 6. **AUTH command** — password challenge on connect for basic access control.

---

_End of document. All code is in [src/](https://file+.vscode-resource.vscode-cdn.net/Users/arghadeep/Desktop/mini-redis-server/src/) and [include/](https://file+.vscode-resource.vscode-cdn.net/Users/arghadeep/Desktop/mini-redis-server/include/). Build with `make`. Connect via `redis-cli -p 6379`._