# Part 1 — Introduction to OOPs
## What Is OOPs? (The Big Picture First)

Imagine you're building a school management system. Without OOPs, you'd write separate variables for every student:
```
string student1_name, student1_dept;
string student2_name, student2_dept;
// ... 500 students = 1000+ lines just for declarations
```

That's unmanageable. OOPs solves this by saying: **"Define what a student looks like once, then stamp out as many as you need."**

> **Object-Oriented Programming (OOPs)** is a way of writing code by modelling real-world entities as **objects** — each object bundles its own data (what it *is*) and functions (what it *can do*) together.

- C++ supports both procedural and OOPs — you choose the style
- Every major language (C++, Java, Python, JS) uses OOPs — learn it once, apply everywhere
- The C++ standard library itself (`vector`, `string`, `map`) is built using OOPs internally

---

## The Two Fundamental Concepts
### Class — The Blueprint
Think of a class like an **architect's blueprint for a house**. The blueprint describes what every house will look like — rooms, doors, windows — but the blueprint itself is not a house. It takes no physical space until you actually build one.

- Defines what **data** (variables) and **behaviour** (functions) each object of that type will have
- **Occupies zero memory by itself** — memory is only allocated when you create an object from it

### Object — The Real Thing

An object is a **concrete, physical instance** of a class — an actual house built from the blueprint.

- Each object gets its **own independent copy** of the data members in memory
- You can create **millions of objects** from the same class — each is independent
- Modifying one object's data does NOT affect another

```cpp
#include <iostream>
#include <string>
using namespace std;

// CLASS — blueprint (no memory yet)
class Teacher {
    public:
        string name;
        string department;
        string subject;
        double salary;

        void changeDepartment(string newDept) {
            department = newDept;   // function that operates on this object's data
        }
};

int main() {
    Teacher t1;   // OBJECT — memory allocated HERE (blueprint → real thing)
    Teacher t2;   // another independent object
    Teacher t3;   // yet another; each gets its own name, dept, subject, salary

    t1.name = "Shraddha";         // set t1's data
    t1.department = "Computer Science";
    t1.subject = "C++";
    t1.salary = 25000;

    cout << t1.name << endl;   // prints: Shraddha
    cout << t2.name << endl;   // prints: "" — t2 has its own separate name (empty)
    return 0;
}
```

---

## Why OOPs? — The "Don't Repeat Yourself" Argument

**Without OOPs:** Every teacher needs separate variables. Adding a new field means editing 100+ lines.
```cpp
string t1_name, t1_dept;   // Teacher 1's data
string t2_name, t2_dept;   // Teacher 2's data — exact same structure, repeated
// For 50 teachers: 100+ declarations. Adding a new field = 50 edits.
```

**With OOPs:** The structure is defined once. Creating a new teacher is one line.
```cpp
Teacher t1, t2, t3;   // all three share the same blueprint
// Need 10,000 teachers? Same code — zero extra declarations.
// Add a new field to the class? One edit, all objects get it automatically.
```

---

## The Four Pillars (Every Interview Asks This)

These are the four core principles of OOPs. Memorise the one-line definitions exactly.

| Pillar | What It Means | Real-World Analogy |
|---|---|---|
| **[[02 - Class, Object, Access Specifiers & Encapsulation#Encapsulation — What It Actually Means\|Encapsulation]]** | Bundle data + methods into one unit (class), and hide sensitive data | A capsule pill — ingredients inside, outer shell is what you interact with |
| **[[06 - Abstraction & Abstract Class#What Is Abstraction? (The ATM Analogy)\|Abstraction]]** | Hide *how* something works internally; expose only *what* it does | ATM machine — you press "withdraw", you don't see the banking logic |
| **[[04 - Inheritance — Modes & Types#What Is Inheritance? (The Intuition First)\|Inheritance]]** | A child class automatically gets properties/methods of a parent class | Child inherits eye colour and height from parents |
| **[[05 - Polymorphism — Overloading, Overriding & Virtual Functions#What Is Polymorphism? (The Intuition First)\|Polymorphism]]** | Same function name behaves differently depending on context | A person behaves differently as an employee, parent, friend — same person, different roles |

> These four pillars are the **most asked OOPs interview questions**. Know them cold.

---

## Real-World to Code Mapping

| Real World | OOPs Concept | Code Example |
|---|---|---|
| Blueprint of a car | Class | `class Car { ... };` |
| Your specific red Toyota (plate: KA-01) | Object (instance) | `Car myCar;` |
| Colour, speed, fuel level | Data members (properties) | `string colour; int speed;` |
| Accelerate, brake, refuel | Member functions (methods) | `void accelerate() { ... }` |
| Amazon product listing | Object | `Product p; p.name="Phone"; p.price=999;` |
| Building from blueprint | Instantiation | `Car myCar;` — the act of creating an object |

---

## `struct` vs `class` — One Critical Difference

```cpp
struct S {
    int x;   // PUBLIC by default — everyone can read/write x
};

class C {
    int x;   // PRIVATE by default — nobody outside can touch x
};
```

- **Convention:** Use `struct` for simple data containers (just data, no behaviour)
- **Convention:** Use `class` for full OOPs entities with data + functions + access control

---

## Terminology Cheat Sheet

| Term | Plain English Meaning |
|---|---|
| **Class** | Blueprint / template that defines a type |
| **Object / Instance** | A concrete, in-memory realisation of a class |
| **Data Member / Attribute** | A variable stored inside an object |
| **Member Function / Method** | A function defined inside a class that operates on its object |
| **Instantiation** | The act of creating an object from a class (`Teacher t1;`) |
| **`.` operator** | Access members of a stack object (`t1.name`) |
| **`->` operator** | Access members of a pointer to an object (`ptr->name`) |

---

# Part 2 — Pointers & Memory

## Why Pointers Matter

Almost everything in C++ OOPs — constructors allocating heap memory, copy constructors, move semantics, virtual functions, polymorphism via base pointers — depends on understanding pointers. If pointers feel shaky, the OOPs topics later will feel like magic tricks instead of logical mechanics.

---

## Variables, Addresses & the `&` Operator

Every variable stored in memory occupies some address. Think of RAM as a giant array of numbered slots — every byte has an address (a number like `0x7FFEE4001`).

The **address-of operator `&`** gives you the address of a variable:

```cpp
int x = 42;

cout << x   << "\n";   // 42         — the value stored in x
cout << &x  << "\n";   // 0x7FFEE... — the address WHERE x lives in memory
```

---

## What Is a Pointer?

A **pointer** is a variable that stores a **memory address** (the address of another variable).

```
Normal variable:   int x = 42;
                   x → stores the VALUE 42

Pointer variable:  int* p = &x;
                   p → stores the ADDRESS of x (e.g., 0x7FFEE4001)
```

```cpp
int x = 42;
int* p = &x;   // p is a pointer to int — it stores x's address
               // read "int*" as "pointer to int"

cout << p    << "\n";   // 0x7FFEE4001 — the address stored in p (address of x)
cout << *p   << "\n";   // 42         — the VALUE at that address (dereferencing)
cout << &p   << "\n";   // 0x7FFEE4008 — the address of p ITSELF (p occupies its own slot in memory)
```

---

## The Dereference Operator `*`

`*p` means "go to the address stored in p, and give me the value there":

```cpp
int x = 42;
int* p = &x;

cout << *p << "\n";   // 42 — reads the value at p's address

*p = 100;             // writes 100 to the address p points to
cout << x  << "\n";   // 100 — x changed! p points to x, so *p = x
```

**Two uses of `*`:**
- In a declaration: `int* p` — "p is a pointer to int"
- In an expression: `*p` — "the value at address p" (dereference)

---

## `nullptr` — The Safe "No Address" Value

A pointer that doesn't point to anything valid should be set to `nullptr`. Never leave a pointer uninitialized — it will contain a garbage address and dereferencing it crashes the program.

```cpp
int* p = nullptr;   // p points to nothing — safe "empty" state

if (p != nullptr) {
    cout << *p;     // only dereference if p is not null
}

// *p = 5;  ← segmentation fault — writing to address 0 is illegal
```

**Rule:** Initialise every pointer to a valid address or `nullptr`. After `delete p;`, always set `p = nullptr`.

---

## Stack vs Heap — Where Variables Live

```
Stack (automatic memory):           Heap (manual memory):
• Local variables                   • Created with 'new'
• Function parameters               • Persists until you 'delete' it
• Auto-freed when scope ends        • YOU are responsible for cleanup
• Fast (just move stack pointer)    • Slower (OS allocator)
• Limited size (~1-8 MB)            • Large (gigabytes available)
```

```cpp
int main() {
    int x = 10;               // STACK — freed automatically when main() ends

    int* p = new int;         // HEAP — allocates one int on heap
    *p = 42;
    cout << *p << "\n";       // 42
    delete p;                 // YOU must free it; forgetting = memory leak
    p = nullptr;              // safety — prevent accidental reuse of freed address

    int* arr = new int[5];    // HEAP — allocates array of 5 ints
    arr[0] = 10;
    arr[1] = 20;
    delete[] arr;             // use delete[] for arrays (not delete!)
    arr = nullptr;
}
```

**Memory leak:** forgetting `delete` — heap block reserved but never freed.
**Dangling pointer:** using a pointer after `delete` — undefined behaviour.

---

## Stack vs Heap — Two Ways to Create Objects

```cpp
Teacher t1;                    // Stack object — auto-destroyed when it goes out of scope

Teacher* t2 = new Teacher();  // Heap object — lives until you explicitly delete it
delete t2;                     // MANDATORY — forgetting this = memory leak
t2 = nullptr;
```

Rule of thumb: prefer stack objects. Only use `new` when the object needs to outlive the current function/block.

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

---

## Pointer Arithmetic

Adding 1 to a pointer moves it forward by **one element size** (not one byte):

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
```cpp
int arr[3] = {1, 2, 3};
// arr == &arr[0]       — same address (arr decays to int*)
// arr[i] == *(arr + i) — two equivalent ways to access element i

// But arr is NOT a pointer — these differ:
sizeof(arr)   // 12 (full array: 3 × 4 bytes) — a real pointer would give 8
&arr          // type is int(*)[3] — pointer to the whole array, NOT int**
```

---

## Pointer to Pointer (`int**`)

Used for 2D arrays on the heap and when a function needs to modify a pointer itself:

```cpp
int x = 42;
int*  p  = &x;    // p points to x
int** pp = &p;    // pp points to p

cout << **pp << "\n";   // 42 — two dereferences: pp → p → x

**pp = 100;             // modifies x through two levels
cout << x << "\n";      // 100
```

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

```cpp
int x = 10, y = 20;

int*             p1 = &x;   // can change both pointer and value
const int*       p2 = &x;   // can change pointer, NOT value  (*p2 = 5 → ERROR)
int* const       p3 = &x;   // can change value, NOT pointer  (p3 = &y → ERROR)
const int* const p4 = &x;   // cannot change either
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
┌──────────────────────┐
│     Stack            │  ← local vars, function call frames (grows downward)
│        ↓             │
│                      │
│        ↑             │
│     Heap             │  ← new/malloc (grows upward)
├──────────────────────┤
│  Static / Global     │  ← static variables, global variables
├──────────────────────┤
│  Code (Text)         │  ← compiled instructions, string literals
└──────────────────────┘
Low address
```

---

## Interview Q&A

**Q: "What is OOPs?"**
A: A programming paradigm that models real-world entities as objects combining state (data) and behaviour (functions), enabling code reusability, modularity, and maintainability.

**Q: "Name the 4 pillars."**
A: Encapsulation, Abstraction, Inheritance, Polymorphism.

**Q: "What is the difference between class and object?"**
A: A class is a blueprint — defines structure but occupies no memory. An object is an instance — the actual thing in memory created from that blueprint.

**Q: "What is a pointer?"**
A: A variable that stores the memory address of another variable. Declared with `*` (e.g., `int* p`). Dereferenced with `*p` to access the value at that address.

**Q: "What is a dangling pointer?"**
A: A pointer that holds the address of memory that has already been freed. Accessing it is undefined behaviour. Fix: set `ptr = nullptr` after `delete`.

**Q: "What is a memory leak?"**
A: Heap memory allocated with `new` that is never freed with `delete`. Accumulates in long-running programs until the process crashes.

**Q: "`delete` vs `delete[]`?"**
A: `delete` for a single object (`new T`). `delete[]` for arrays (`new T[n]`). Using the wrong one is undefined behaviour.

**Q: "What is `nullptr`?"**
A: A C++11 keyword for a null pointer — safer than `NULL` (which is `0` and causes ambiguity with integer overloads).

**Q: "Stack vs heap?"**
A: Stack: automatic, fast, freed at scope end, limited size. Heap: manual (`new`/`delete`), larger, lives until explicitly freed.

**Q: "What does `p->member` mean?"**
A: Shorthand for `(*p).member` — dereference the pointer to get the object, then access its member.
