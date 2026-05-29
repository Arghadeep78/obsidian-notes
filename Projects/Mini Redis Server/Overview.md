# Mini Redis Server — Complete Concepts & Interview Guide

A from-scratch Redis-compatible in-memory data store implemented in C++17. Built as a systems programming portfolio project demonstrating TCP networking, concurrent client handling, custom protocol parsing, and persistent storage — all without external libraries.

---

##  Table of Contents

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

## 1. Project Overview

### What is it?

A TCP server that speaks the Redis wire protocol (RESP) and implements a subset of real Redis commands. You can connect to it with `redis-cli` exactly as you would connect to a real Redis server — it is protocol-compatible.

### What does it implement?

- 29 Redis commands across 3 data types: **strings**, **lists**, **hashes** (28 unique + UNLINK as alias for DEL)
- TTL / key expiration (`EXPIRE`)
- Persistence to disk (saves every 5 minutes + on shutdown)
- Concurrent multi-client handling via threads
- Graceful shutdown on `Ctrl+C`

### Tech stack

|Layer|Technology|
|---|---|
|Language|C++17|
|Networking|POSIX sockets (`sys/socket.h`)|
|Concurrency|`std::thread`, `std::mutex`, `std::atomic`|
|Protocol|RESP (Redis Serialization Protocol)|
|Storage|STL containers (`unordered_map`, `vector`)|
|Persistence|Custom text-based file format|
|Build|`make` (g++ with `-std=c++17 -Wall -pthread -MMD -MP -O2`)|

---

## 2. Architecture

### High-Level Data Flow

```
CLIENT (redis-cli or any TCP client)
  │
  │  TCP connection on port 6379
  ▼
RedisServer::run()
  ├── listen() on TCP socket
  ├── accept() → new client_socket
  └── spawn detached std::thread per client
        │
        │  while recv() > 0
        ▼
  RedisCommandHandler::processCommand()
        ├── parseRespCommand()  →  [cmd, arg1, arg2, ...]
        ├── dispatch to handler function (e.g. handleSet)
        └── return RESP-encoded response string
              │
              ▼
        RedisDatabase::getInstance()
              ├── lock_guard<mutex> db_mutex
              ├── operate on kv_store / list_store / hash_store
              └── return result
        │
        │  send(response) back to client
        ▼
CLIENT receives RESP response
```

### Component Diagram

```
┌────────────────────────────────────────────────────┐
│                 Main Thread                        │
│  main() → load dump → spawn persist thread        │
│  → RedisServer::run() (blocks on accept loop)     │
└─────────────────┬──────────────────────────────────┘
                  │  accept() → spawn thread
                  ▼
┌────────────────────────────────────────────────────┐
│  Client Thread (detached, one per connection)      │
│  recv → processCommand → send                     │
└──────────────────────┬─────────────────────────────┘
                       │  all threads share this
                       ▼
┌────────────────────────────────────────────────────┐
│  RedisDatabase  (Singleton)                        │
│  ┌──────────┐  ┌───────────────┐  ┌────────────┐  │
│  │ kv_store │  │  list_store   │  │ hash_store │  │
│  └──────────┘  └───────────────┘  └────────────┘  │
│  ┌──────────────────────────────────────────────┐  │
│  │  expiry_map  (TTL tracking)                  │  │
│  └──────────────────────────────────────────────┘  │
│  Protected by: std::mutex db_mutex                 │
└──────────────────────┬─────────────────────────────┘
                       │  every 5 min + on shutdown
                       ▼
              dump.my_rdb  (disk)

┌────────────────────────────────────────────────────┐
│  Background Persist Thread (detached)              │
│  infinite loop: sleep(300s) → dump()              │
└────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────┐
│  SIGINT Handler                                    │
│  Ctrl+C → shutdown() → dump() → close socket     │
└────────────────────────────────────────────────────┘
```

---

## 3. File Structure

```
mini-redis-server/
├── main.cpp                     # Entry point: startup, background persist, signal setup
├── include/
│   ├── RedisServer.h            # TCP server class declaration
│   ├── RedisDatabase.h          # In-memory data store class declaration (Singleton)
│   └── RedisCommandHandler.h    # RESP parser + command dispatcher declaration
├── src/
│   ├── RedisServer.cpp          # TCP socket, accept loop, client threads, SIGINT
│   ├── RedisDatabase.cpp        # All data stores, mutex locking, TTL, dump/load
│   └── RedisCommandHandler.cpp  # RESP parser + 29 command dispatch entries
├── test_all.sh                  # Integration test script (uses redis-cli)
├── Makefile                     # Build: g++ -std=c++17 -Wall -pthread -MMD -MP -O2
└── dump.my_rdb                  # Persisted data file (auto-created on first dump)
```

### Responsibilities at a Glance

|File|Role|
|---|---|
|`main.cpp`|Wires everything together; owns startup/shutdown orchestration|
|`RedisServer.cpp`|Network I/O only — no business logic|
|`RedisDatabase.cpp`|All state — no networking, no protocol|
|`RedisCommandHandler.cpp`|Protocol + dispatch — bridge between network and storage|

---

## 4. Data Structures & Why

### Primary Storage (inside RedisDatabase)

#### `std::unordered_map<string, string> kv_store` — String values

- **Why `unordered_map`**: O(1) average insert/lookup/delete. Perfect for cache-style workloads.
- **Stores**: everything set via `SET key value`.
- **Trade-off vs `map`**: O(1) vs O(log N); `map` would give sorted iteration but at a cost.

#### `std::unordered_map<string, vector<string>> list_store` — Lists

- **Why `vector` for list elements**: Simple, contiguous memory, O(1) random access and `push_back`.
- **Trade-off**: `LPUSH` (prepend) is O(N) because vector shifts all elements. Real Redis uses a doubly-linked list / `quicklist` for O(1) on both ends.
- **Implication**: For long lists, `LPUSH` is slow. `RPUSH`/`RPOP` are fast.

#### `std::unordered_map<string, unordered_map<string, string>> hash_store` — Hashes

- **Why nested `unordered_map`**: Outer map = O(1) key lookup. Inner map = O(1) field lookup. Models the Redis hash data type naturally.
- **Memory**: Each hash key independently allocates its own field map.

#### `std::unordered_map<string, steady_clock::time_point> expiry_map` — TTL tracking

- **Why separate map**: Only keys _with_ a TTL are tracked here — keeps this sparse. No need to check every key for expiry.
- **Why `steady_clock`**: Monotonic clock — not affected by system clock adjustments (NTP, DST, etc.).
- **Eviction strategy**: Lazy — expired keys are only removed when they are accessed (see [Section 8](https://file+.vscode-resource.vscode-cdn.net/Users/arghadeep/Desktop/mini-redis-server/specifics.md#8-ttl--expiration)).

#### `std::mutex db_mutex` — Thread safety

- Single coarse-grained mutex protecting all three stores.
- Every public method in `RedisDatabase` wraps its body in `std::lock_guard<std::mutex> lock(db_mutex)`.
- RAII: lock is automatically released when `lock_guard` goes out of scope.

### Transient Structures (in-flight only)

|Structure|Where used|Purpose|
|---|---|---|
|`vector<string>`|`parseRespCommand()`|Tokenized command arguments|
|`ostringstream`|Handler functions|Building RESP response strings|
|`vector<pair<string,string>>`|`hmset()`|Bulk field-value pairs for HMSET|

---

## 5. Concurrency Model

### Threading Architecture: One Thread Per Client

```
Main thread
  └── accept() loop
        ├── accept() → client_fd
        ├── std::thread([client_fd]{ ... }).detach()
        ├── accept() → another client_fd
        └── ...

Detached thread (per client):
  while recv() > 0:
    processCommand(input) → send(response)
  close(client_fd)
  // thread exits
```

- **Thread creation**: A new `std::thread` is spawned for every accepted connection.
- **Detached**: `.detach()` is called immediately — the main thread does not track or join client threads.
- **Why detached**: Avoids blocking the accept loop. The main thread should always be ready to accept new connections.
- **Lifecycle**: Thread lives until the client disconnects (`recv()` returns 0 or an error).

### Synchronization: Single Global Mutex

```cpp
// Every method in RedisDatabase looks like this:
bool RedisDatabase::set(const string& key, const string& value) {
    std::lock_guard<std::mutex> lock(db_mutex);  // Acquired here
    // ... do work ...
}  // Lock automatically released here (RAII)
```

- **Coarse-grained locking**: One mutex guards all three maps.
- **No deadlock risk**: Only one lock exists; no lock ordering issues.
- **Bottleneck**: Under high concurrency, all clients serialize on `db_mutex`. For a portfolio project this is acceptable. In production, sharded locks or lock-free structures would be used.

### `std::atomic<bool> running`

- The accept loop checks `while (running)`.
- Set to `false` by the SIGINT handler on shutdown.
- `std::atomic` ensures the write from the signal handler is visible to the main thread without a mutex.

### Thread Safety Properties

|Property|Status|
|---|---|
|No data races|✅ — all reads/writes locked by `db_mutex`|
|No deadlocks|✅ — single lock, no nesting|
|Per-command atomicity|✅ — each command holds lock for its entire duration|
|Multi-command transactions|❌ — not implemented (no MULTI/EXEC)|
|Graceful drain on shutdown|❌ — `exit()` kills detached threads mid-flight|

---

## 6. RESP Protocol

### What is RESP?

RESP (Redis Serialization Protocol) is the wire protocol used by Redis. It encodes commands and responses as ASCII text with type prefixes and length headers. It is human-readable and easy to parse.

### Two Parsing Modes

The server supports both modes, auto-detected:

#### Mode 1: Inline (plain text)

```
PING\r\n
SET foo bar\r\n
```

Parsing: split on whitespace.

#### Mode 2: RESP Array Format (used by redis-cli and all Redis clients)

```
*3\r\n       ← 3 elements follow
$3\r\n       ← next string is 3 bytes
SET\r\n
$3\r\n
foo\r\n
$3\r\n
bar\r\n
```

Parsing in `parseRespCommand()`:

1. Read `*N` → `N` elements expected
2. For each element: read `$LEN` then read exactly `LEN` bytes
3. Return `vector<string>` of tokens

Validation:

- Reject `N <= 0` or `N > 1024`
- `try/catch` around `stoi()` for malformed headers
- If input is truncated mid-frame, return empty vector (caller handles as error)

### Response Encoding

Every handler function returns a RESP-encoded `std::string`:

|Response type|Format|Example|
|---|---|---|
|Simple string|`+<text>\r\n`|`+OK\r\n`|
|Error|`-<msg>\r\n`|`-Error: key not found\r\n`|
|Integer|`:<n>\r\n`|`:42\r\n`|
|Bulk string|`$<len>\r\n<bytes>\r\n`|`$5\r\nhello\r\n`|
|Null (nil)|`$-1\r\n`|(missing key)|
|Array|`*<count>\r\n<elements>`|`*2\r\n$3\r\nfoo\r\n$3\r\nbar\r\n`|

### Command Dispatch Flow

```
processCommand(raw_input)
  ├── tokens = parseRespCommand(raw_input)
  ├── cmd = tokens[0].toUpperCase()    ← case-insensitive
  ├── if (cmd == "SET") return handleSet(tokens, db)
  ├── if (cmd == "GET") return handleGet(tokens, db)
  ├── ... (29 if-else branches)
  └── return "-Error: Unknown command\r\n"
```

---

## 7. All Supported Commands

### General (3 commands)

|Command|Syntax|Returns|Notes|
|---|---|---|---|
|`PING`|`PING`|`+PONG`|Liveness check|
|`ECHO`|`ECHO <msg>`|`+<msg>`|Echo back message|
|`FLUSHALL`|`FLUSHALL`|`+OK`|Clears all stores and expiry_map|

### String / Key-Value (7 commands)

|Command|Syntax|Returns|Notes|
|---|---|---|---|
|`SET`|`SET key value`|`+OK`|Clears any existing TTL on the key|
|`GET`|`GET key`|`$<len>\r\n<value>` or `$-1`|Calls `purgeExpired()` before read|
|`DEL`|`DEL key`|`:1` or `:0`|Deletes from any store; clears TTL|
|`UNLINK`|`UNLINK key`|`:1` or `:0`|Alias for DEL (same implementation)|
|`KEYS`|`KEYS`|`*N\r\n...`|Returns all keys across all 3 stores|
|`TYPE`|`TYPE key`|`+string`, `+list`, `+hash`, or `+none`|Checks which store contains the key|
|`EXPIRE`|`EXPIRE key seconds`|`+OK` or error|Stores absolute time_point = now + seconds|
|`RENAME`|`RENAME old new`|`+OK` or error|Moves key + value + TTL to new name|

### List (9 commands)

|Command|Syntax|Returns|Notes|
|---|---|---|---|
|`LPUSH`|`LPUSH key v1 [v2...]`|`:<new_length>`|Prepends to front (O(N) — vector shift)|
|`RPUSH`|`RPUSH key v1 [v2...]`|`:<new_length>`|Appends to back (O(1) — `push_back`)|
|`LPOP`|`LPOP key`|`$<value>` or `$-1`|Removes and returns first element|
|`RPOP`|`RPOP key`|`$<value>` or `$-1`|Removes and returns last element|
|`LLEN`|`LLEN key`|`:<length>`|Count of elements|
|`LGET`|`LGET key`|`*N\r\n...`|Returns entire list as array (non-standard; real Redis uses `LRANGE`)|
|`LINDEX`|`LINDEX key index`|`$<value>` or `$-1`|Supports negative indices (-1 = last)|
|`LSET`|`LSET key index value`|`+OK` or error|Sets element at index; supports negatives|
|`LREM`|`LREM key count value`|`:<removed_count>`|count>0: from head; count<0: from tail; count=0: remove all|

### Hash (9 commands)

|Command|Syntax|Returns|Notes|
|---|---|---|---|
|`HSET`|`HSET key field value`|`:1` (new) or `:0` (updated)|Creates hash if key doesn't exist|
|`HGET`|`HGET key field`|`$<value>` or `$-1`|Returns nil if key or field missing|
|`HMSET`|`HMSET key f1 v1 [f2 v2...]`|`+OK`|Bulk-set multiple fields atomically|
|`HGETALL`|`HGETALL key`|`*2N\r\n...`|Flat array: `[f1, v1, f2, v2, ...]`|
|`HKEYS`|`HKEYS key`|`*N\r\n...`|All field names|
|`HVALS`|`HVALS key`|`*N\r\n...`|All field values|
|`HLEN`|`HLEN key`|`:<count>`|Number of fields|
|`HEXISTS`|`HEXISTS key field`|`:1` or `:0`|Check field existence|
|`HDEL`|`HDEL key field`|`:1` or `:0`|Removes field; returns 1 if it existed|

---

## 8. TTL / Expiration

### How it Works

1. `EXPIRE key 60` computes `expiry_time = steady_clock::now() + 60 seconds` and stores it in `expiry_map[key]`.
2. Before **certain** operations, `purgeExpired()` is called. It iterates `expiry_map`, checks each entry against `now()`, and **deletes** expired keys from all stores.
3. `purgeExpired()` does **NOT** acquire `db_mutex` itself — it assumes the calling method already holds the lock (all callers do via `lock_guard`). Acquiring the same mutex twice from the same thread would deadlock.
4. If a key is `SET` again after `EXPIRE`, its TTL is cleared (`expiry_map.erase(key)`).

**Critical limitation:** `purgeExpired()` is only called from 6 methods: `get()`, `keys()`, `type()`, `del()`, `expire()`, and `rename()`. All list and hash read operations (`lget`, `llen`, `hget`, `hgetall`, `hkeys`, `hvals`, `hlen`, `hexists`, `lpop`, `rpop`, etc.) do **NOT** call `purgeExpired()`. This means an expired list or hash key can still be read and returned — a known bug in the implementation.

### Eviction Strategy: Lazy / On-Access

```
String read (e.g. GET) arrives:
  ├── lock_guard acquired
  ├── purgeExpired() called  ← only string/key-metadata ops do this
  │     └── scan expiry_map → remove any expired keys from all stores
  └── proceed with read

List/Hash read (e.g. LGET, HGET) arrives:
  ├── lock_guard acquired
  └── proceed with read  ← purgeExpired() is NOT called (known bug)
```

- **Advantage**: No dedicated background thread for eviction; simple to implement.
- **Disadvantage**: Expired keys that are never read again stay in memory. List/hash reads bypass expiry checks entirely.
- **Real Redis**: Uses both lazy eviction (on access, for all types) AND a background thread that samples random keys periodically to proactively remove expired ones.

### TTL Persistence Limitation

The current dump format does not save expiry times. On restart, all TTLs are lost — keys that were about to expire now live forever.

---

## 9. Persistence

### Overview

Data is saved to `dump.my_rdb` in a custom text format. This is conceptually similar to Redis's RDB (snapshot) persistence.

### File Format

```
K <key> <value>
L <key> <item1> <item2> <item3>
H <key> <field1>:<val1> <field2>:<val2>
```

Example:

```
K username alice
K session abc123
L queue task1 task2 task3
H user:1 name:Bob age:25 city:NYC
```

### When Saves Happen

|Trigger|When|Where in code|
|---|---|---|
|Startup load|Before first client connects|`main.cpp`: `db.load("dump.my_rdb")`|
|Background save|Every 300 seconds (5 minutes)|`main.cpp`: detached thread|
|Graceful shutdown|On `SIGINT` (Ctrl+C)|`RedisServer::shutdown()`|
|Run() exit|When accept loop ends|`RedisServer::run()` final lines|

### Dump Implementation

```cpp
bool RedisDatabase::dump(const string& filename) {
    lock_guard<mutex> lock(db_mutex);
    ofstream ofs(filename, ios::binary);
    if (!ofs) return false;

    for (auto& [key, val] : kv_store)
        ofs << "K " << key << " " << val << "\n";

    for (auto& [key, items] : list_store) {
        ofs << "L " << key;
        for (auto& item : items) ofs << " " << item;
        ofs << "\n";
    }

    for (auto& [key, fields] : hash_store) {
        ofs << "H " << key;
        for (auto& [f, v] : fields) ofs << " " << f << ":" << v;
        ofs << "\n";
    }
    return true;
}
```

### Load Implementation

```cpp
bool RedisDatabase::load(const string& filename) {
    lock_guard<mutex> lock(db_mutex);
    ifstream ifs(filename, ios::binary);
    if (!ifs) return false;

    kv_store.clear(); list_store.clear(); hash_store.clear();

    string line;
    while (getline(ifs, line)) {
        istringstream ss(line);
        char type; ss >> type;
        if (type == 'K') { /* parse key value */ }
        if (type == 'L') { /* parse key + space-separated items */ }
        if (type == 'H') { /* parse key + field:value pairs */ }
    }
    return true;
}
```

### Persistence Limitations

- **No atomic writes**: If the process crashes mid-dump, the file is partially written and corrupted.
- **No checksums**: Corrupt lines are silently skipped or misparse.
- **Space-delimited**: Values containing spaces break the parser.
- **No TTL persistence**: Expiry times are lost on restart.
- **No compression**: Large datasets produce large text files.

---

## 10. Server Lifecycle

### Startup Sequence

```
1. main() parses argv[1] → port (default 6379, range 1–65535)

2. RedisDatabase::getInstance().load("dump.my_rdb")
   └── Singleton created on first call (C++11 thread-safe static init)
   └── If file exists: restores all key-value/list/hash data

3. RedisServer server(port)
   └── Constructor: sets globalServer = this
   └── setupSignalHandler(): registers SIGINT → signalHandler()

4. Background persist thread spawned (detached):
   └── infinite: sleep(300s) → db.dump("dump.my_rdb")

5. server.run()  [blocks here]
   └── Creates TCP socket (AF_INET, SOCK_STREAM)
   └── setsockopt(SO_REUSEADDR) ← allows fast restart without "address in use"
   └── bind(INADDR_ANY, port) ← listen on all interfaces
   └── listen(backlog=10)
   └── Enter accept loop (see below)
```

### Accept Loop

```
while (running) {
    client_fd = accept(server_fd, nullptr, nullptr)

    if (client_fd < 0) {
        if (running) print error
        else break  // shutdown was called, stop accepting
    }

    thread([client_fd]() {
        char buf[1024];
        while (true) {
            int n = recv(client_fd, buf, sizeof(buf) - 1, 0);  // reads up to 1023 bytes
            if (n <= 0) break;            // disconnect or error
            string req(buf, n);
            string resp = cmdHandler.processCommand(req);
            if (send(client_fd, resp.c_str(), resp.size(), 0) < 0) break;
        }
        close(client_fd);
    }).detach();
}

// After loop exits:
db.dump("dump.my_rdb");  // final save
```

### Shutdown Sequence (Ctrl+C)

```
1. OS delivers SIGINT to process
2. signalHandler() called (registered at startup)
3. globalServer->shutdown():
   a. running = false          ← signals accept loop to stop
   b. db.dump("dump.my_rdb")  ← final persistence save
   c. close(server_fd)         ← unblocks accept() call in main thread
4. exit(SIGINT)                ← process terminates
   └── All detached client threads are killed (no graceful drain)
```

---

## 11. Error Handling

### Five Layers of Error Handling

**Layer 1 — RESP Parse Errors** (`parseRespCommand()`):

```cpp
// Malformed header
try { numElements = stoi(line.substr(1)); }
catch (exception&) { return {}; }  // empty = error

// Out-of-range element count
if (numElements <= 0 || numElements > 1024) return {};

// Truncated frame
if (pos + len > input.size()) break;
```

Caller gets empty `vector<string>` → returns `-Error: Empty command\r\n`.

**Layer 2 — Argument Validation** (each handler):

```cpp
static string handleSet(const vector<string>& tokens, RedisDatabase& db) {
    if (tokens.size() < 3)
        return "-Error: SET requires key and value\r\n";
    // ...
}
```

**Layer 3 — Numeric Argument Parsing**:

```cpp
try {
    int seconds = stoi(tokens[2]);
    // use seconds...
} catch (const exception&) {
    return "-Error: Invalid expiration time\r\n";
}
```

**Layer 4 — Socket I/O**:

```cpp
int n = recv(client_fd, buf, sizeof(buf), 0);
if (n <= 0) break;  // client disconnected or error → exit thread cleanly

if (send(client_fd, ...) < 0) break;  // send failed → exit thread
```

**Layer 5 — File I/O**:

```cpp
ofstream ofs(filename);
if (!ofs) return false;  // caller logs the failure
```

### Error Response Format

All errors use RESP error format: `-Error: <description>\r\n`

This is displayed by `redis-cli` as `(error) Error: <description>`.

---

## 12. Design Decisions & Trade-offs

### 1. Singleton Pattern for RedisDatabase

**Decision**: `RedisDatabase::getInstance()` returns a static local variable.

**Why**: Ensures exactly one database instance across all threads. C++11 guarantees thread-safe initialization of static local variables. Every component (handlers, server) accesses the same instance without passing it around.

**Trade-off**: Hard to test in isolation (can't create two separate DB instances for tests).

---

### 2. Coarse-Grained Mutex (Single Lock)

**Decision**: One `std::mutex db_mutex` protects all three stores.

**Why**: Dead-simple. No lock ordering to think about. Zero risk of deadlock.

**Trade-off**: All concurrent clients serialize through this one lock. If Client A is doing a slow `LREM` on a 100k-element list, every other client waits.

**What real Redis does**: Single-threaded event loop (no mutex needed at all) + async I/O. Writes are always serial. The I/O multiplexing (epoll) provides concurrency of connections without parallelism in the DB.

---

### 3. One Thread Per Client

**Decision**: Spawn a new `std::thread` per accepted connection.

**Why**: Straightforward mental model. Each client gets its own stack, its own recv/send loop.

**Trade-off**: Thread creation is expensive. At 10,000 simultaneous clients, spawning 10k threads kills performance and memory. Real servers use:

- **Thread pool** with a fixed number of workers
- **Event-driven I/O** (epoll/kqueue) with non-blocking sockets — handle thousands of clients in one thread

---

### 4. Lazy TTL Eviction

**Decision**: Expired keys are only removed when accessed (on-read eviction).

**Why**: No background thread needed. Simple.

**Trade-off**: A key with a 1-second TTL that's never read again sits in memory indefinitely. Under high TTL usage, memory grows without bound until keys are accessed.

---

### 5. `std::vector` for List Elements

**Decision**: Lists are `vector<string>`, not `deque<string>` or a linked list.

**Why**: `vector` is the simplest STL sequence container and most cache-friendly for reads.

**Trade-off**: `LPUSH` is O(N) because all elements shift right. For large lists, this is a performance cliff. A `std::deque` would give O(1) on both ends with minimal code change.

---

### 6. Text-Based Persistence Format

**Decision**: Human-readable space-delimited text instead of binary RDB.

**Why**: Much easier to implement and debug. You can open the file in a text editor.

**Trade-off**: Breaks with spaces in keys/values. No checksums. Slow to parse. Large file size. No TTL persistence.

---

### 7. Fixed 1024-byte Receive Buffer

**Decision**: `char buf[1024]` per `recv()` call, no dynamic reassembly.

**Why**: Keeps the networking code simple.

**Trade-off**: Any command longer than 1024 bytes is silently truncated. In RESP format, even a moderately large `HMSET` with long values can exceed this. A real implementation would use a ring buffer or grow-on-demand buffer to accumulate a complete RESP frame before parsing.

---

### 8. Case-Insensitive Commands

**Decision**: `std::transform(cmd, ..., ::toupper)` before dispatching.

**Why**: Redis is case-insensitive (`set` == `SET`). Doing it at dispatch time means every handler only needs to handle uppercase.

---

## 13. Known Limitations

|Limitation|Impact|Fix|
|---|---|---|
|`LPUSH`/`LPOP` are O(N)|Slow for large lists|Use `std::deque`|
|Single mutex|Bottleneck under high concurrency|Per-shard or per-key locking|
|1024-byte recv buffer|Silently truncates large commands|Dynamic buffer / frame reassembly|
|No TTL persistence|Expirations lost on restart|Write timestamps to dump file|
|`load()` does not clear `expiry_map`|Stale TTL entries could linger after a reload|Clear `expiry_map` alongside the three data stores in `load()`|
|`purgeExpired()` not called in list/hash reads|Expired list/hash keys can still be returned by LGET, HGET, etc.|Call `purgeExpired()` at the start of every read method|
|Text dump format|Breaks with spaces in keys/values|Binary format or JSON|
|No atomic writes to disk|Risk of file corruption on crash mid-dump|Write to temp file, then rename atomically|
|No MULTI/EXEC|No transactions|Implement transaction state per client|
|Detached threads on shutdown|In-flight requests killed by `exit()`|Thread pool with join on shutdown|
|No AUTH|No access control|Password challenge on connect|
|No SELECT|Single keyspace|Multiple databases via separate maps|
|One-thread-per-client|Doesn't scale past ~1000 clients|Event loop (epoll) + thread pool|
|Lazy-only TTL eviction|Expired-but-unread keys stay in memory indefinitely|Background reaper thread sampling random keys|

---

## 14. Build & Run

### Build

```bash
make
# Produces: ./my_redis_server
```

Compile flags: `g++ -std=c++17 -Wall -pthread -MMD -MP -O2`

### Run

```bash
./my_redis_server          # Listen on default port 6379
./my_redis_server 6380     # Listen on port 6380
```

On startup you'll see:

```
Database Loaded From dump.my_rdb
Redis Server Listening On Port 6379
```

(If no dump file exists: `No dump found or load failed; starting with an empty database.`)

### Connect

```bash
redis-cli -p 6379          # Standard Redis CLI
redis-cli -p 6379 PING     # Should return PONG
```

### Shutdown

`Ctrl+C` — triggers SIGINT handler, performs final save, closes socket.

---

## 15. Testing

### test_all.sh

A bash integration test script that pipes Redis commands to `redis-cli`. It covers all 29 commands.

```bash
./test_all.sh   # Requires server already running on port 6379
```

Works by stripping comment lines with `sed '/^#/d'` and feeding the rest to `redis-cli` via stdin.

**Coverage**:

- ✅ All 29 commands
- ✅ Basic correctness (expected values returned)
- ✅ TTL setting (`EXPIRE`)
- ❌ Concurrent access
- ❌ TTL expiry after time passes
- ❌ Persistence across restart
- ❌ Error cases (wrong argument counts, type mismatches)
- ❌ Large payloads (>1024 bytes)

---

## 16. Interview Q&A

Prepared answers to common interview questions about this project.

---

**Q: Walk me through what happens when a client runs `SET foo bar`.**

> 1. `redis-cli` encodes the command as a RESP array: `*3\r\n$3\r\nSET\r\n$3\r\nfoo\r\n$3\r\nbar\r\n` and sends it over TCP.
> 2. The server's accept loop has already spawned a dedicated thread for this client. That thread's `recv()` call returns the bytes.
> 3. `processCommand()` is called. `parseRespCommand()` detects the `*` prefix and parses out three tokens: `["SET", "foo", "bar"]`.
> 4. The command is uppercased (`SET`) and dispatched to `handleSet()`.
> 5. `handleSet()` calls `db.set("foo", "bar")`. Inside `set()`, a `lock_guard` acquires `db_mutex`, stores `kv_store["foo"] = "bar"`, and erases any existing entry in `expiry_map["foo"]` (clearing old TTL).
> 6. `handleSet()` returns `"+OK\r\n"`.
> 7. The client thread calls `send()` with this response.
> 8. `redis-cli` receives `+OK` and prints `OK`.

---

**Q: How does your server handle multiple clients at the same time?**

> Each accepted connection spawns a detached `std::thread`. These threads all share the `RedisDatabase` singleton. Access to the database is protected by a single `std::mutex` — each operation acquires a `lock_guard` which ensures operations don't interleave. So clients truly run concurrently on separate threads, but database operations are serialized through the mutex.

---

**Q: What's the biggest performance bottleneck and how would you fix it?**

> The single global mutex is the main bottleneck — all concurrent clients serialize on every database operation. The fix depends on the target scale:
> 
> Short-term: Replace the coarse lock with reader-writer locks (`std::shared_mutex`) — multiple concurrent readers, exclusive writers. This is a safe 2x–10x improvement for read-heavy workloads.
> 
> Long-term: Move to an event-driven architecture like real Redis — a single-threaded event loop with `epoll`, non-blocking sockets, and an internal command queue. This eliminates mutex contention entirely since the DB is only ever touched from one thread.

---

**Q: How does your TTL implementation work? What are its weaknesses?**

> `EXPIRE key 60` stores `steady_clock::now() + 60s` in `expiry_map`. `purgeExpired()` is called before certain operations — specifically `get()`, `keys()`, `type()`, `del()`, `expire()`, and `rename()` — and scans `expiry_map` for entries past their expiry, deleting them from all stores. It does NOT acquire `db_mutex` itself; it assumes the caller already holds it.
> 
> Weaknesses: (1) Lazy eviction — expired keys that are never read again stay in memory forever. (2) List and hash read operations (`LGET`, `HGET`, etc.) do NOT call `purgeExpired()`, so an expired list or hash key can still be returned — this is a bug. (3) TTLs are not saved to disk, so they're lost on restart. (4) `purgeExpired()` is O(E) where E = number of keys with TTLs — could be slow if many keys have TTLs.
> 
> Fix: Call `purgeExpired()` at the start of every read method. Add a background thread that samples random TTL keys every 100ms and evicts expired ones — this is what real Redis does.

---

**Q: Why did you use `unordered_map` instead of `map`?**

> `unordered_map` gives O(1) average-case lookup, insert, and delete via hashing. `map` gives O(log N) via a balanced BST. For a key-value cache, access patterns are hash-lookup-heavy, not range-scan-heavy, so O(1) is the better fit. The downside is that `unordered_map` doesn't support ordered iteration (e.g., `KEYS *` returns keys in arbitrary order), but Redis's `KEYS` command makes no ordering guarantees anyway.

---

**Q: How does your persistence work? Is it safe?**

> Data is dumped to `dump.my_rdb` in a human-readable text format on a background thread every 5 minutes and on shutdown. On startup, the file is loaded to restore state.
> 
> It's not crash-safe. If the process crashes mid-dump, the file is partially written and the next load will parse garbage or miss data. A safe approach is: write to a temp file (`dump.tmp`), fsync it, then rename it over the live file atomically. This is what Redis's BGSAVE does. I'd add that as an improvement.

---

**Q: What design pattern does `RedisDatabase` use and why?**

> It's a Singleton using the Meyer's Singleton pattern — a static local variable in `getInstance()`. C++11 guarantees this initialization is thread-safe. The rationale: there should be exactly one in-memory database, shared across all connection-handling threads. Rather than passing a reference everywhere, the Singleton gives any code a clean way to get the one database instance. The downside is that it makes unit testing harder — you can't easily create an isolated DB instance per test.

---

**Q: How would you scale this to handle 100,000 concurrent connections?**

> The current one-thread-per-client model breaks down around 1,000–10,000 threads due to OS scheduling overhead and memory (each thread uses ~8MB stack). I'd redesign the networking layer to use:
> 
> 1. **epoll (Linux)** or **kqueue (macOS)** for I/O multiplexing — a single thread can monitor thousands of sockets for readability/writability with O(1) event detection.
> 2. **A thread pool** (e.g., N = CPU cores) where worker threads pull ready socket events from a queue.
> 3. **Non-blocking sockets** so no thread blocks on a slow client.
> 
> The database layer would also need sharded locks or a lock-free structure to avoid the mutex becoming the bottleneck.

---

**Q: What would you add if you had two more weeks?**

> Priority order:
> 
> 1. **MULTI/EXEC transactions** — queues commands, executes them atomically
> 2. **Safe persistence** (write to temp + rename) + TTL serialization
> 3. **Dynamic receive buffer** to handle commands > 1024 bytes
> 4. **`std::deque` for lists** to make LPUSH O(1)
> 5. **TTL background reaper thread** to reclaim memory from expired-but-unread keys
> 6. **AUTH command** for basic access control

---

_This document covers the full implementation. All code references are in `src/` and `include/`. The project compiles with `make` and connects via `redis-cli -p 6379`._