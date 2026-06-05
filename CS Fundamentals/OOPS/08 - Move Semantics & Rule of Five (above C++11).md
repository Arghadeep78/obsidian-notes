# Part 1 — lvalue/rvalue, Move Constructor & Move Assignment

## The Problem — Why Move Semantics Exist

Imagine you have a large vector of 1 million integers, and you want to "give" it to another function. Before C++11, the only option was **copying** via [[03 - Constructors, this Pointer, Copy Constructor, Shallow & Deep Copy & Destructor#Copy Constructor — Making a Clone|copy constructors]] — allocate 1 million slots of new memory and copy every element across. That's slow.

But what if the original vector is a **temporary** that's about to be destroyed anyway? You're copying data from something you're about to throw away. That's wasteful — like photocopying a document just to shred the original.

**Move semantics** (C++11) say: "Instead of copying, just *steal the resources* — take ownership of the existing memory."

```
Copy = "make me a clone"    → source unchanged, new memory allocated, O(n) cost
Move = "give me your guts"  → source emptied, just pointer swap, O(1) cost
```

---

## lvalue vs rvalue — The Foundation

You must understand this before move semantics makes sense.

> **lvalue** = has a name, has a persistent memory address, can appear on the *left* of `=`
> **rvalue** = temporary, no name, about to be destroyed, cannot appear on the *left* of `=`

```cpp
int x = 5;         // x is lvalue (has a name, has an address)
                   // 5 is rvalue (a literal — temporary, no name)
string s = "hi";   // s is lvalue
                   // "hi" is rvalue (string literal)
string t = s;      // s is lvalue → COPY happens (s still alive, t gets a new copy)
string u = move(s); // std::move(s) casts s to rvalue → MOVE happens (s emptied)
```

| Expression | lvalue or rvalue? |
|---|---|
| Named variable `x` | lvalue |
| `42`, `3.14`, `"hello"` | rvalue |
| Return value of a function (temporary) | rvalue |
| `std::move(x)` | rvalue (forced cast — tells compiler "treat x as temporary") |
| `x + y` (arithmetic result) | rvalue |

**Rvalue reference** (`&&`) is a new type in C++11 that binds specifically to rvalues, enabling move operations.

---
## `std::move` — What It Actually Is

- `std::move(x)` **does not move anything**.
- It is just a **cast**: converts `x` from an lvalue to an rvalue reference (`T&&`).
```
std::move(x);   // "You may steal resources from x"
```
- The **actual move** happens when a move constructor or move assignment operator is called.
```
string s2 = std::move(s1);   // move constructor runs
```
#### After Moving From an Object
- Source object is **valid but unspecified**.
- Object is still alive.
- Destructor is **not** called.
- You can:
    - destroy it
    - assign to it
    - call valid member functions
```
s1 = "new value";   // OK
```
#### Do NOT Assume Its Value
```
cout << s1;   // Legal
```
but the result is unspecified (often empty for `string`).

---

# Part 2 — Rule of Five, Rule of Zero & Smart Pointers

## Rule of Five (C++11)

In C++03, the "Rule of Three" said: if your class manages heap resources, define the [[03 - Constructors, this Pointer, Copy Constructor, Shallow & Deep Copy & Destructor#Destructor — The "Cleanup Man"|destructor]], [[03 - Constructors, this Pointer, Copy Constructor, Shallow & Deep Copy & Destructor#Copy Constructor — Making a Clone|copy constructor]], and copy assignment operator.

C++11 added move operations, making it the **Rule of Five**:

> **If you define ANY ONE of these five, you almost certainly need all five:**

| # | Special Member Function | When It's Called |
|---|---|---|
| 1 | Destructor `~T()` | Object lifetime ends |
| 2 | Copy Constructor `T(const T&)` | `T b(a)`, `T b = a`, pass/return by value |
| 3 | Copy Assignment `T& operator=(const T&)` | `b = a` when b already exists |
| 4 | **Move Constructor** `T(T&&)` | `T b(move(a))`, return temp from function |
| 5 | **Move Assignment** `T& operator=(T&&)` | `b = move(a)` |

- **Destructor missing** → Resource leak.
- **Copy constructor missing** → Shallow copy → double delete.
- **Copy assignment missing** → Shallow copy, leaks, double delete.
- **Move constructor missing** → Expensive copies instead of cheap moves.
- **Move assignment missing** → Expensive copy assignment instead of move.

---
## Rule of Zero — The Modern Idiomatic Approach

If your class **only contains** RAII-managed members (`unique_ptr`, `shared_ptr`, `string`, `vector`), you can define **none of the five** and the compiler-generated defaults handle everything correctly:

```cpp
class ModernResource {
    unique_ptr<int[]> data;   // RAII — unique_ptr handles delete[] automatically
    size_t size;

public:
    ModernResource(size_t n) : size(n), data(make_unique<int[]>(n)) {}
    // No destructor needed — unique_ptr's destructor frees the array
    // No copy constructor — unique_ptr is non-copyable (prevents accidental copies)
    // No copy assignment — same reason
    // Move constructor — compiler generates it, uses unique_ptr's move
    // Move assignment — compiler generates it, uses unique_ptr's move
};
```

**Rule of Zero:** Prefer composing your class out of RAII types so you need zero special members.

---

## Interview Q&A

**Q: "What is move semantics?"**
A: The ability to transfer ownership of resources (heap memory, file handles, etc.) from a temporary object to a new one without copying. Achieved via move constructor and move assignment using rvalue references (`&&`). Makes returning large objects from functions essentially free.

**Q: "What is `std::move`?"**
A: A cast — not a move. It casts an lvalue to an rvalue reference, signalling to the compiler that the move constructor/assignment should be selected. The source object is left in a valid but unspecified (usually empty) state after the move.

**Q: "What is the Rule of Five?"**
A: If you define any of: destructor, copy constructor, copy assignment, move constructor, or move assignment — you almost certainly need all five. Applies to any class managing raw resources directly.

**Q: "lvalue vs rvalue?"**
A: lvalue has a name and a persistent address (can appear on left of `=`). rvalue is a temporary with no name that's about to be destroyed (cannot appear on left of `=`).

**Q: "When is move constructor called vs copy constructor?"**
A: Move constructor is called when the source is an rvalue (temporary or `std::move(x)`). Copy constructor is called when the source is a named lvalue.

**Q: "What is `noexcept` on move operations?"**
A: Marks the operation as non-throwing. The STL (e.g., `std::vector` during reallocation) will only use move instead of copy if the move is `noexcept` — otherwise it falls back to (safe but slower) copy.

**Q: "What is the Rule of Zero?"**
A: If your class only uses RAII-managed members, define none of the five special members — the compiler generates correct defaults automatically.

---

## Special Topic — Smart Pointers

### The Problem with Raw Pointers

Raw pointers (`new`/`delete`) require you to manually track ownership — who allocated this memory and who is responsible for freeing it. Forget to `delete` → memory leak. `delete` twice → double free. Exception thrown before `delete` → leak.

Smart pointers wrap a raw pointer in an object whose **destructor automatically calls `delete`** — so cleanup is guaranteed regardless of how the scope exits (normal return, exception, early return). This is RAII applied to heap memory.

```cpp
#include <memory>   // all smart pointers live here
```

---

### `unique_ptr` — Single, Exclusive Ownership

`unique_ptr<T>` owns the object exclusively. **One and only one** `unique_ptr` can own the resource at a time. When the `unique_ptr` goes out of scope, the resource is automatically freed.

```cpp
#include <memory>
#include <iostream>
using namespace std;

class Dog {
public:
    string name;
    Dog(string n) : name(n) { cout << name << " created\n"; }
    ~Dog() { cout << name << " destroyed\n"; }
    void bark() { cout << name << ": Woof!\n"; }
};

int main() {
    unique_ptr<Dog> p1 = make_unique<Dog>("Bruno");  // preferred way to create
    p1->bark();           // Bruno: Woof! — use -> just like a raw pointer
    cout << p1->name;     // Bruno

    // unique_ptr is NON-COPYABLE — ownership is exclusive
    // unique_ptr<Dog> p2 = p1;  ← ❌ COMPILE ERROR: copy is deleted

    // Transfer ownership with move
    unique_ptr<Dog> p2 = move(p1);   // p2 now owns Bruno, p1 is empty (nullptr)
    p2->bark();           // Bruno: Woof!
    // p1->bark();        ← ❌ p1 is now nullptr — crash

}  // p2 goes out of scope → destructor auto-called → "Bruno destroyed"
   // No delete needed!
```

**Use `unique_ptr` when:** One owner. Simple ownership. Most common smart pointer.

---

### `shared_ptr` — Shared Ownership with Reference Counting

`shared_ptr<T>` allows **multiple pointers to share ownership** of the same object. It keeps an internal **reference count** — how many `shared_ptr`s currently point to the object. When the count drops to zero, the object is automatically deleted.

```cpp
#include <memory>
using namespace std;

int main() {
    shared_ptr<Dog> p1 = make_shared<Dog>("Max");  // ref count = 1
    cout << p1.use_count() << "\n";   // 1

    {
        shared_ptr<Dog> p2 = p1;      // copy is ALLOWED — both share ownership
        cout << p1.use_count() << "\n";  // 2 — two owners now
        p2->bark();   // Max: Woof!
    }   // p2 goes out of scope → ref count drops to 1 — object NOT destroyed yet

    cout << p1.use_count() << "\n";   // 1
    p1->bark();   // Max: Woof! — still alive
}   // p1 goes out of scope → ref count = 0 → "Max destroyed"
```

**Use `shared_ptr` when:** Multiple parts of the code genuinely share ownership and it's unclear who should be "last" to free the resource.

---

### `weak_ptr` — Non-Owning Observer

`weak_ptr<T>` holds a **non-owning reference** to an object managed by `shared_ptr`. It does NOT increase the reference count. Used to **break circular references** (the main pitfall of `shared_ptr`).

**The circular reference problem:**
```cpp
struct Node {
    shared_ptr<Node> next;   // Node A owns Node B
    // shared_ptr<Node> prev; // Node B also owns Node A → ref count never reaches 0 → LEAK!
};
```

**Fix with `weak_ptr`:**
```cpp
struct Node {
    shared_ptr<Node> next;   // owning forward link
    weak_ptr<Node>   prev;   // NON-owning backward link — no ref count increase
};
```

To use a `weak_ptr`, you must first `lock()` it — this checks if the object still exists and returns a `shared_ptr` (or empty if already destroyed):

```cpp
shared_ptr<Dog> sp = make_shared<Dog>("Rex");
weak_ptr<Dog>   wp = sp;              // wp observes sp — ref count still 1

if (auto locked = wp.lock()) {        // lock() returns shared_ptr if object still alive
    locked->bark();                   // Rex: Woof!
} else {
    cout << "Object already destroyed\n";
}

sp.reset();                           // explicitly destroy — ref count → 0 → "Rex destroyed"
if (auto locked = wp.lock()) {
    // won't enter here
} else {
    cout << "Object already destroyed\n";  // ← prints this
}
```

**Use `weak_ptr` when:** You need to observe an object without extending its lifetime (caches, back-pointers in trees/graphs, observer pattern).

---

### Smart Pointer Comparison

| | `unique_ptr` | `shared_ptr` | `weak_ptr` |
|---|---|---|---|
| Ownership | Exclusive (1 owner) | Shared (N owners) | None (observer) |
| Copyable? | ❌ No | ✅ Yes | ✅ Yes |
| Movable? | ✅ Yes | ✅ Yes | ✅ Yes |
| Ref count? | No | Yes | No (uses shared's count) |
| Overhead | Zero (same as raw ptr) | Small (ref count alloc) | Small |
| Use for | Default choice | Shared ownership | Breaking cycles |

### Always Prefer `make_unique` / `make_shared`

```cpp
// BAD — two separate allocations, exception-unsafe
shared_ptr<Dog> p(new Dog("Bruno"));

// GOOD — single allocation, exception-safe, clearer intent
auto p = make_shared<Dog>("Bruno");
auto q = make_unique<Dog>("Max");
```

**Why `make_shared` is faster — the single-allocation advantage:**

`shared_ptr` needs two things on the heap: the object itself AND a *control block* (which stores the reference count, weak count, and deleter). `new Dog(...)` followed by `shared_ptr<Dog>(...)` performs **two separate heap allocations** — one for the `Dog`, one for the control block. `make_shared<Dog>(...)` allocates **both in a single block** — halving the number of heap calls, improving cache locality (the object and its ref count sit next to each other in memory), and eliminating the window where an exception between the `new` and the `shared_ptr` constructor could leak the object.

### Interview One-Liners

**Q: "What is a smart pointer?"**
A: An RAII wrapper around a raw pointer that automatically calls `delete` when it goes out of scope, preventing memory leaks and double-frees.

**Q: "`unique_ptr` vs `shared_ptr`?"**
A: `unique_ptr` has exclusive ownership — non-copyable, zero overhead. `shared_ptr` has shared ownership via reference counting — copyable, small overhead. Default to `unique_ptr`; use `shared_ptr` only when ownership is genuinely shared.

**Q: "What is `weak_ptr`?"**
A: A non-owning observer of a `shared_ptr`-managed object. Doesn't affect reference count. Used to break circular references. Must `lock()` before use to check if the object is still alive.
