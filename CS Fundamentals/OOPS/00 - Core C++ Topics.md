# Part 1 - Important Keywords
## Operator Overloading (Extra)

> Redefine what built-in operators (`=`, `+`, `==`, `<<`, etc.) do for your custom types.

```cpp
class Vector2D {
public:
    double x, y;
    Vector2D(double x = 0, double y = 0) : x(x), y(y) {} //same name valid
    Vector2D operator+(const Vector2D& other) const {
        return Vector2D(x + other.x, y + other.y);
    }

    bool operator==(const Vector2D& other) const {
        return (x == other.x && y == other.y);
    }

    // << must be non-member (ostream on left) but needs private access → friend
    friend ostream& operator<<(ostream& os, const Vector2D& v) {
        os << "(" << v.x << ", " << v.y << ")";
        return os;
    } //& return same object type (cout, file stream, etc.) without copy and Streams are not copyable
};

int main() {
    Vector2D v1(1, 2), v2(3, 4);
    Vector2D v3 = v1 + v2;        // calls operator+
    cout << v3 << "\n";           // (4, 6) — calls operator<<
    cout << (v1 == v2) << "\n";   // 0 (false)
}
```

**Operators you CANNOT overload:** `::` (scope), `.` (member access), `.*`, `?:` (ternary), `sizeof`, `.*` (member pointer access).

**Can overload:** `+ - * / % = == != < > <= >= ++ -- [] () << >> -> && || !`

---

## `explicit` Keyword — Prevent Surprise Conversions

> `explicit` on a **single-argument constructor** tells the compiler: "never use this constructor for *implicit* type conversions."

```cpp
class Radius {
public:
    double value;
    explicit Radius(double v) : value(v) {}
};

void drawCircle(Radius r) { cout << r.value; }

int main() {
    drawCircle(5.0);           // ERROR: can't implicitly convert double → Radius
    drawCircle(Radius(5.0));   // ✅ explicit — intention is clear
}
```

Without `explicit`, `drawCircle(5.0)` would silently convert `5.0` to a `Radius` object — which can hide bugs where you accidentally pass the wrong value.

---

## `mutable` Keyword — Modify in `const` Functions

> `mutable` marks a member that can be changed even inside a `const` member function.

Use it only for **internal bookkeeping** that doesn't affect the object's logical state:

**const**: This function promises *not* to *modify* the *object*.

```cpp
class Cache {
    string data;
    mutable int accessCount = 0;   // mutable — tracking reads doesn't change "data"
public:
    Cache(string d) : data(d) {}

    string getData() const {
        accessCount++;   // ✅ allowed because accessCount is mutable
        return data;
    }

    int getAccessCount() const { return accessCount; }
};
```

**Rule:** Use `mutable` only for things like access counters, caches, and mutexes — things that change internally but don't change the observable value of the object.

---

## Special Topic — `= default` and `= delete`

### The Problem Without These

Before C++11, managing compiler-generated functions was messy:

- To prevent copying, you had to declare a copy constructor `private` and leave it unimplemented (a hack).
- To use a compiler default constructor alongside a custom constructor, you had to write an empty body `{}` manually, which is less efficient than compiler-generated code.

C++11 introduced `= default` and `= delete` to handle these scenarios cleanly.

### `= default` — Explicitly Requests Compiler-Generated Version

- Instructs the compiler to generate its standard, optimized version of a special member function.
#### Why use it

Writing any custom constructor causes the compiler to stop generating the default constructor automatically. `= default` restores it.

error otherwise:
```cpp
class Student {
public:
	int age;
	Student(int age) {
		this->age = age;
	}
};
Student s1(20); // OK
Student s2; // Error
```

#### `default` example:

```cpp
class MyClass {
public:
    MyClass() = default;                        // "compiler, please generate the default constructor"
    MyClass(const MyClass&) = default;          // generate default copy constructor
    MyClass& operator=(const MyClass&) = default; // generate default copy assignment
    ~MyClass() = default;                       // generate default destructor
};
```

**Why use `= default` instead of just not writing it?**
- Makes intent explicit and visible in code
- Sometimes needed to re-enable a suppressed default.

| You Declare           | Copy Constructor | Copy Assignment | Move Constructor | Move Assignment |
| --------------------- | ---------------- | --------------- | ---------------- | --------------- |
| Nothing               | ✅                | ✅               | ✅                | ✅               |
| Destructor only       | ✅                | ✅               | ❌                | ❌               |
| Copy constructor only | ✅                | ✅               | ❌                | ❌               |
| Copy assignment only  | ✅                | ✅               | ❌                | ❌               |
| Move constructor only | ❌                | ❌               | ✅                | ✅               |
| Move assignment only  | ❌                | ❌               | ✅                | ✅               |

### `= delete` — Explicitly Disable a Function

```cpp
class NonCopyable {
public:
    NonCopyable() = default;
    NonCopyable(const NonCopyable&) = delete;  // copying is DISABLED
    NonCopyable& operator=(const NonCopyable&) = delete; // copy assignment DISABLED (need to disable seperately)
    // Move is still allowed
    NonCopyable(NonCopyable&&) = default;
    NonCopyable& operator=(NonCopyable&&) = default;
};

NonCopyable a;
NonCopyable b(a);   // ❌ COMPILE ERROR: use of deleted function
NonCopyable c = a;  // ❌ COMPILE ERROR
NonCopyable d(move(a)); // ✅ move is fine
```

**Common real-world use:** File handles, mutexes, database connections — resources that must not be copied. `std::unique_ptr` uses `= delete` on its copy constructor internally.

`= delete` can also block unwanted implicit conversions:
```cpp
class OnlyInt {
public:
    void process(int x) { cout << x; }
    void process(double) = delete; // prevent silent int-double conversion
    void process(bool)   = delete;
};
OnlyInt o;
o.process(5);     // ✅ int — fine
o.process(3.14);  // ❌ COMPILE ERROR — double overload is deleted
o.process(true);  // ❌ COMPILE ERROR — bool overload is deleted
```

---

## Special Topic — `std::initializer_list` & Uniform Initialization

### Uniform Initialization (`{}` syntax — C++11)

Before C++11, there were multiple inconsistent ways to initialise things. C++11 introduced `{}` (brace initialization) as one universal syntax that works everywhere:

```cpp
int x = 5;       // old style — assignment initialization
int y(5);        // old style — direct initialization
int z{5};        // C++11 — uniform/brace initialization ✅ preferred

int arr[] = {1, 2, 3};   // old
int arr2[] {1, 2, 3};    // C++11 brace init

// Works for objects too:
Teacher t1("Shraddha", "CS", "C++", 25000);   // calls matching constructor
Teacher t1{"Shraddha", "CS", "C++", 25000};   // also valid
```

**Bonus: `{}` prevents narrowing conversions**(type cast from more to less information) (a common source of silent bugs):
```cpp
int a(3.14);    // old style: silently truncates to 3 — no warning
int b{3.14};    // ❌ COMPILE ERROR: narrowing conversion not allowed in {}
                // {} is stricter — protects from accidental data loss
```

### `std::initializer_list` — Constructing from a List of Values

`std::initializer_list<T>` lets a constructor accept a **variable number of values of the same type**, written with `{}` *only*. This is how `std::vector` and other containers support `{1, 2, 3, 4, 5}` construction:

```cpp
class NumberSet {
    vector<int> data;
public:
    NumberSet(std::initializer_list<int> values) {
        for (int v : values)
            data.push_back(v);
    } 
};
int main() {
    NumberSet s{10, 20, 30, 40, 50};   // calls the initializer_list constructor not on ()
}
```

**Important: `std::initializer_list` constructor takes priority over other constructors** when `{}` is used: (not to be confused with [[#2 ways to write a constructor|member initializer list constructor]])

```cpp
class Tricky {
public:
    Tricky(int n, int val){
	    cout << "Regular ctor: " << n << " copies of " << val << "\n";
    }
    Tricky(initializer_list<int> lst){
	    cout << "initializer_list ctor, size=" << lst.size() << "\n";
	}
};

Tricky a(3, 5);    // Regular ctor: 3 copies of 5   — () picks regular
Tricky b{3, 5};    // initializer_list ctor, size=2  — {} picks initializer_list!
```

This is a well-known C++11 gotcha: `vector<int> v(5, 0)` creates 5 zeros, but `vector<int> v{5, 0}` creates a 2-element vector `[5, 0]`.

---

## Extra Topic — POD Types

### What Is a POD Type?

**POD (Plain Old Data)** = a type that is compatible with *C*. It's both an [[#3 ways to write a constructor|Aggregate]] AND all members of same access specifiers, no base classes, user defines assignment(=).

>All PODs are aggregates, but not all aggregates are PODs.

```cpp
// POD type — layout-compatible with C structs
struct Packet {
    int id;
    float value;
    char flags;
};
// Can safely use memcpy, sizeof, pass over network, write to binary file, etc.
```

**Why POD matters:**
- Safe to copy with `memcpy` (bit-by-bit copy is valid)
- Safe to zero-initialize with `memset`
- Used in binary file I/O, network protocols, interoperability with C
- `sizeof` gives predictable layout

```cpp
Packet p = {42, 3.14f, 'A'};// Safe for binary ops:
Packet p2;
memcpy(&p2, &p, sizeof(Packet));   // valid for POD — bitwise copy is correct
```

**Non-POD (fails one or more POD rules):**
```cpp
class Teacher {
    string name;           // string has a constructor/destructor — NOT POD
    virtual void foo();    // virtual function — NOT POD
};
```

**Interview one-liner:** "A POD type is a simple C-compatible struct with no constructors, destructors, virtual functions, or non-public members. Safe for `memcpy` and binary I/O."

---

# Part 2 — Pointers & Memory

## Why Pointers Matter
Most C++ OOP concepts—dynamic memory allocation, copy/move semantics, virtual functions, and polymorphism through base-class pointers -> are built on pointers.
## Variables, Addresses & the `&` Operator
Every variable stored in memory occupies some address. Think of RAM as a giant array of numbered slots — every byte has an address (a number like `0x7FFEE4001`).
The **address-of operator `&`** gives you the address of a variable:

```cpp
int x = 42;
int int* p = &x;
int* a = nullptr;   // p points to nothing — safe "empty" state
cout << x   << "\n";    // 42         — the value stored in x
cout << &x  << "\n";   // 0x7FFEE... — the address WHERE x lives in memory
cout << *p << "\n";   // 42 — reads the value at p's address
cout << *a << endl;   // Segmentation fault
```
## What Is a Pointer?
A **pointer** is a variable that stores a **memory address** (the address of another variable).
`int* p = &x`

```
Normal variable:   int x = 42;
                   x → stores the VALUE 42
Pointer variable:  int* p = &x;
                   p → stores the ADDRESS of x (e.g., 0x7FFEE4001)
```

## The Dereference Operator `*`
`*p` means "go to the address stored in p, and give me the value there":
**Two uses of `*`:**
- In a declaration: `int* p` — "p is a pointer to int"
- In an expression: `*p` — "the value at address p" (dereference)
## `nullptr` — The Safe "No Address" Value

Always initialize pointers to `nullptr` if they don't point to a valid object. Uninitialized pointers contain garbage addresses, and dereferencing them causes undefined behavior.

> Dereferencing it gives Segmentation fault

**Rule:** Initialise every pointer to a valid address or `nullptr`. After `delete p;`, always set `p = nullptr`.

---

## Stack vs Heap — Two Ways to Create Objects

| Stack (Automatic Memory)                            | Heap (Dynamic Memory)                         |
| --------------------------------------------------- | --------------------------------------------- |
| Local variables & objects                           | Dynamic data                                  |
| Automatic memory management (no `new`)              | Manual management (`new` / `delete`)          |
| Freed automatically when scope ends                 | Must be freed manually                        |
| Faster allocation/deallocation (move stack pointer) | Slower allocation/deallocation (OS allocator) |
| Limited size (~1–8 MB typical)                      | Much larger (often GBs)                       |

```cpp
int main() {
    int x = 10;            // STACK — freed automatically when main() ends
    int* p = new int;      //  HEAP — allocates one int on heap
    delete p;              // YOU must free it; forgetting = memory leak
    p = nullptr;           // safety: prevent accidental reuse
    int* arr = new int[5];    // HEAP — allocates array of 5 ints
    arr[0] = 10;
    arr[1] = 20;
    delete[] arr;             // use delete[] for arrays (not delete!)
    arr = nullptr;
}
```

**Memory leak:** heap memory block allocated with `new` is **not fred** using `delete` — heap block reserved but never freed. makes the memory **unreachable** but still **occupied** i.e. memory '**wasted**'
**Dangling pointer:** using a pointer after `delete` — undefined behaviour.

**Rule of thumb:**
- Normal variables/objects → Stack, `new` → Heap.
- prefer stack objects. Only use `new` when the object needs to outlive the current function/block.
>**`delete` vs `delete[]`** → use (`delete arr[]`) for arrays , (`delete p`) for single objects

---
## Pointers to Objects — Dot vs Arrow

Stack object → use `.` | Heap (pointer) → use `->`

```cpp
class Dog {
public:
    string name;
    void bark() { cout << name << " says: Woof!\n"; }
};

int main() {
    Dog d1;              // stack
    d1.name = "Bruno";   // dot operator
    d1.bark();

    Dog* d2 = new Dog(); // heap
    d2->name = "Max";    // arrow operator — same as (*d2).name
    d2->bark();
    delete d2;
    d2 = nullptr;
}
```

`ptr->member` is exactly equivalent to `(*ptr).member` — just shorter.
## Pointer Arithmetic

Adding 1 to a pointer moves it forward by **one element size** (not one byte):

a pointer to a int, *int\** is typically *8 bytes*, *int* is *4 bytes*, *int*** is also 8 bytes

```cpp
int arr[5] = {10, 20, 30, 40, 50};
int* p = arr;            // p points to arr[0]

cout << *p       << "\n";   // 10 — arr[0]
cout << *(p + 1) << "\n";   // 20 — arr[1] (moved by sizeof(int) = 4 bytes)
cout << *(p + 2) << "\n";   // 30 — arr[2]

p++;
cout << *p << "\n";          // 20 — p now points to arr[1]
```

**Array name decays to a pointer** to its first element in most expressions:
`arr` *decays to* `&arr[0]` (an `int*`).
```cpp
int arr[3] = {1, 2, 3};
// arr == &arr[0]       — same address (arr decays to int*)
// arr[i] == *(arr + i) — two equivalent ways to access element i
// But arr is NOT a pointer — these differ:
sizeof(arr)   // 12 (full array: 3 × 4 bytes) — a real pointer would give 8
&arr          // type: int(*)[3] — pointer to the whole array, NOT int**
arr           // type: int* decays to &arr[0]
```

#### 1. `int arr[] = {0,0,0,0};` (Stack Array)

- **What `arr` is:** A fixed-size array type (`int[4]`).
- **Decay behavior:** When used in expressions, the array name **decays** into a pointer (`int*`) pointing to `&arr[0]`.
- **Evaluation of `arr[1]`**: The array name `arr` decays into a pointer i.e. `&arr[0]`, and the compiler evaluates `arr[1]` exactly as `*(arr + 1)`.
#### 2. `int* arr = new int[5];` (Dynamic Pointer)

- **What `arr` is:** It is already a primitive pointer variable (`int*`) allocated on the stack that stores a memory address.
- **Decay behavior:** No decay happens because it is already a pointer (arr already stores &arr\[0\]). It simply holds the address of the first element of the heap-allocated array.
- **Evaluation of `arr[1]`**: Because `arr` is already a pointer, the compiler directly evaluates `arr[1]` as `*(arr + 1)` without any decay, shifting the address by `1 * sizeof(int)` bytes on the heap.

- **Value of `arr + 1`:** Directly yields the memory address of the second element (`&arr[1]`) in both cases.

	refer [[03 - Constructors, this Pointer, Copy Constructor, Shallow & Deep Copy & Destructor#Pointer Allocation Behavior| Pointer Allocation Behavior]] for more details
## Pointer to Pointer (`int**`)

Used for 2D arrays when a function needs to modify a pointer itself:

```cpp
int x = 42;
int*  p  = &x;    // p points to x
int** pp = &p;    // pp points to p
cout << **pp << "\n";   // 42 — two dereferences: pp → p → x
**pp = 100;             // modifies x through two levels
cout << x << "\n";      // 100
```
all above var are on *stack*

Dynamic 2D array using `int**`:
```cpp
int rows = 3, cols = 4;
int** matrix = new int*[rows];
for (int i = 0; i < rows; i++)
    matrix[i] = new int[cols];

matrix[1][2] = 99;

for (int i = 0; i < rows; i++)
    delete[] matrix[i];
delete[] matrix;
```

---

## `void*` — The Generic Pointer

Holds the address of any type, but cannot be dereferenced directly:

```cpp
int x = 42;
void* p = &x;        // ✅ any address fits in void*

// *p = 5;           ← ❌ ERROR: compiler doesn't know the type

int* ip = static_cast<int*>(p);   // must cast before use
cout << *ip << "\n";               // 42
```

Used in C APIs like `memcpy` and `malloc` for type-agnostic memory operations.

---

## `const` with Pointers — Quick Reference

(Covered in depth in [[09 - const Correctness & Type Casting in C++ OOPs#`const` with Pointers — The Four Combinations|const Correctness]])
*read right->left*

```cpp
int x = 10, y = 20;

// p1 is a pointer to an int
int* p1 = &x;             // Pointer: p1 = &y;  ✔ | Value: *p1 = 30;  ✔
// p2 is a pointer to a const int
const int* p2 = &x;       // Pointer: p2 = &y;  ✔ | Value: *p2 = 30;  ❌
// p3 is a const pointer to an int
int* const p3 = &x;       // Pointer: p3 = &y;  ❌ | Value: *p3 = 30;  ✔
// p4 is a const pointer to a const int
const int* const p4 = &x; // Pointer: p4 = &y;  ❌ | Value: *p4 = 30;  ❌
```

---

## Function Pointers — Pointing to Code

```cpp
int add(int a, int b) { return a + b; }
int sub(int a, int b) { return a - b; }

int (*op)(int, int) = add;   // op points to a function: (int, int) → int
cout << op(3, 4) << "\n";    // 7

op = sub;
cout << op(3, 4) << "\n";    // -1
```

The compiler uses function pointers internally to build the vtable for virtual functions.

Uses:
1. Passing Functions as Arguments
2. array of function pointers instead of nested if else
> int (\*operations[3])(int, int) = { add, subtract, multiply };

>A **vtable** (virtual method table) is a compiler-generated lookup table of function pointers used to resolve virtual function calls at runtime for dynamic polymorphism.

---

## Common Pointer Bugs — Know These Cold

| Bug                  | What It Is                      | Example                             |
| -------------------- | ------------------------------- | ----------------------------------- |
| **Memory leak**      | `new` without `delete`          | `int* p = new int; /* no delete */` |
| **Dangling pointer** | Using pointer after `delete`    | `delete p; cout << *p;`             |
| **Double free**      | `delete`-ing same address twice | `delete p; delete p;`               |
| **Null dereference** | Dereferencing `nullptr`         | `int* p = nullptr; *p = 5;`         |
| **Buffer overflow**  | Writing past allocated bounds   | `int* p = new int[3]; p[5] = 1;`    |
| **Wild pointer**     | Uninitialized pointer used      | `int* p; *p = 5;`                   |

**Prevention:** After every `delete` → set `ptr = nullptr`. Before every dereference → check `if (ptr != nullptr)`.

---

## Memory Layout — The Full Picture

```
High address
┌──────────────────┐
│      Stack       │  ← local vars, function call frames (grows downward)
│        ↓         │    grow down
│                  │
│        ↑         │    grow up
│      Heap        │  ← new/malloc (grows upward)
├──────────────────┤
│  Static / Global │  ← static variables, global variables (compile time    |                  |                                        allocation)
├──────────────────┤
│  Code (Text)     │  ← compiled instructions, string literals
└──────────────────┘
Low address
```

