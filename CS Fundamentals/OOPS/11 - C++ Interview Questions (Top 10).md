# C++ Most Important Interview Questions

---

## **1. Pointers and Its Types (Dangling Pointer)**

A **pointer** is a variable that stores the memory address of another variable.

```cpp
int x = 10;
int* p = &x;   // p holds the address of x
cout << *p;    // dereference → prints 10
```

**Types of Pointers:**

- **Null Pointer** — points to nothing; `int* p = nullptr;`
- **Void Pointer** — generic pointer, no type; `void* p;` — must be cast before dereferencing
- **Wild Pointer** — declared but never initialised; holds a garbage address (undefined behaviour)
- **Dangling Pointer** — *points to memory that has already been freed*
- **Smart Pointers** (`unique_ptr`, `shared_ptr`, `weak_ptr`) — RAII wrappers that auto-delete

**Dangling Pointer — the most asked:**
```cpp
int* p = new int(5);
delete p;      // memory freed
// p still holds the old address — it is now a dangling pointer
*p = 10;       // undefined behaviour — DO NOT do this
p = nullptr;   // fix: null it out immediately after delete
```

> *After `delete`, always set the pointer to `nullptr`. Dereferencing a dangling pointer is undefined behaviour and a common source of crashes and security vulnerabilities.*

---

## **2. Memory Allocation, Memory Layout of C++ Programs, Memory Leak**

**Memory Layout of a C++ Program:**

```
High Address
┌──────────────────┐
│   Command-line   │  (argc, argv)
├──────────────────┤
│      Stack       │  ← grows downward
│   (local vars,   │
│   function frames│
│   return addrs)  │
├──────────────────┤
│        ↓         │
│      (gap)       │
│        ↑         │
├──────────────────┤
│      Heap        │  ← grows upward (new / malloc)
├──────────────────┤
│  BSS Segment     │  uninitialised global/static vars (zeroed)
├──────────────────┤
│  Data Segment    │  initialised global/static vars
├──────────────────┤
│  Text Segment    │  compiled machine code (read-only)
└──────────────────┘
Low Address
```

**Memory Allocation:**

| Type | Mechanism | Lifetime | Where |
|---|---|---|---|
| Static | Global / `static` keyword | Entire program run | Data / BSS segment |
| Automatic | Local variables | Until scope ends | Stack |
| Dynamic | `new` / `delete` | Until explicitly freed | Heap |

**Memory Leak:**
- Happens when heap memory allocated with `new` is *never freed* with `delete`
- In long-running programs, leaked memory accumulates → eventual crash or OOM

```cpp
void leak() {
    int* p = new int(42);
    // forgot delete p; → 4 bytes leaked every call
}
```

Fix: always `delete` what you `new`, or use smart pointers (`unique_ptr` auto-deletes).

---

## **3. Stack vs Heap Memory**

| | **Stack** | **Heap** |
|---|---|---|
| Management | Automatic (compiler) | Manual (`new`/`delete`) or smart ptrs |
| Speed | *Very fast* — just moves stack pointer | Slower — allocator finds free block |
| Size | Small (~1–8 MB) | Large (limited by RAM) |
| Lifetime | Freed when scope exits | Lives until explicitly freed |
| Fragmentation | None | Can fragment over time |
| Overflow | Stack overflow (deep recursion) | Out-of-memory |
| Use case | Local vars, function calls | Large data, objects with dynamic lifetime |

```cpp
int a = 5;         // stack — auto freed when function returns
int* p = new int;  // heap  — must manually delete p
```

---

## **4. C++ Program Execution: Preprocessor → Compiler → Assembler → Linker → Loader**

```
Source (.cpp)
      │
      ▼
┌─────────────┐
│ Preprocessor│  handles #include, #define, #ifdef → produces expanded .cpp
└─────────────┘
      │
      ▼
┌─────────────┐
│  Compiler   │  syntax/semantic checks → generates Assembly (.s)
└─────────────┘
      │
      ▼
┌─────────────┐
│  Assembler  │  converts Assembly → Object file (.o / .obj)
└─────────────┘
      │
      ▼
┌─────────────┐
│   Linker    │  combines .o files + libraries → executable (.exe / a.out)
└─────────────┘
      │
      ▼
┌─────────────┐
│   Loader    │  OS loads executable into RAM, sets up stack/heap, starts main()
└─────────────┘
```

- **Preprocessor** — text substitution; no type checking
- **Compiler** — type checking, optimisation, code generation
- **Assembler** — machine-specific translation
- **Linker** — resolves external symbols (e.g., `printf` from libc); static vs dynamic linking
- **Loader** — OS component; performs address relocation, loads shared libraries

---

## **5. Call by Value vs Call by Reference vs Call by Pointer**

| | **By Value** | **By Reference** | **By Pointer** |
|---|---|---|---|
| Syntax | `void f(int x)` | `void f(int& x)` | `void f(int* x)` |
| Copy made? | Yes — independent copy | No — alias to original | No — address copied |
| Modifies original? | No | *Yes* | Yes (via `*x`) |
| Can be null? | N/A | No | Yes |
| Overhead | Copy cost | None | Minimal (address copy) |
| Use when | Don't need to modify | Modify or avoid copy | Optional param / arrays |

```cpp
void byVal(int x)  { x = 99; }         // original unchanged
void byRef(int& x) { x = 99; }         // original changed
void byPtr(int* x) { *x = 99; }        // original changed via dereference

int a = 5;
byVal(a);   // a = 5
byRef(a);   // a = 99
byPtr(&a);  // a = 99
```

> *Prefer references over pointers for modifiable parameters — cleaner syntax and can't be null.*

---

## **6. Compile-time vs Runtime Polymorphism**

**Polymorphism** = same interface, different behaviour.

| | **Compile-time (Static)** | **Runtime (Dynamic)** |
|---|---|---|
| Resolved at | Compile time | Runtime |
| Mechanism | Function overloading, templates, operator overloading | Virtual functions + inheritance |
| Binding | *Early binding* | *Late binding* (vtable lookup) |
| Performance | Faster — no vtable overhead | Slight overhead (pointer indirection) |
| Flexibility | Less flexible | More flexible — base pointer → any derived |

```cpp
// Compile-time — resolved by signature
void print(int x)   { cout << x; }
void print(float x) { cout << x; }

// Runtime — resolved by actual object type at runtime
class Shape { public: virtual void draw() { cout << "Shape"; } };
class Circle : public Shape { public: void draw() override { cout << "Circle"; } };

Shape* s = new Circle();
s->draw();  // prints "Circle" — NOT "Shape"
```

---

## **7. Function Overloading vs Overriding**

| | **Overloading** | **Overriding** |
|---|---|---|
| Where | Same class (or scope) | Parent → Child class |
| Signature | *Different* parameters | *Same* signature |
| Polymorphism | Compile-time | Runtime |
| `virtual` needed? | No | Yes (in parent) |
| `override` keyword | No | Recommended (C++11) |

```cpp
// Overloading — same name, different params
int add(int a, int b)       { return a + b; }
double add(double a, double b) { return a + b; }

// Overriding — child redefines parent's virtual function
class Animal { public: virtual void speak() { cout << "..."; } };
class Dog : public Animal { public: void speak() override { cout << "Woof"; } };
```

> *Overloading is resolved by the compiler. Overriding is resolved at runtime via the vtable.*

---

## **8. Virtual Functions**

A **virtual function** is a member function declared with `virtual` in the base class, allowing derived classes to override it with runtime dispatch.

**How it works — vtable:**
- The compiler creates a **vtable** (virtual function table) for every class with virtual functions
- Each object carries a hidden **vptr** pointing to its class's vtable
- On a virtual call via base pointer, the vptr is followed → correct function called at runtime

```cpp
class Base {
public:
    virtual void show() { cout << "Base"; }
    virtual ~Base() {}   // always virtual destructor if using polymorphism
};
class Derived : public Base {
public:
    void show() override { cout << "Derived"; }
};

Base* b = new Derived();
b->show();   // "Derived" — runtime dispatch via vtable
delete b;    // calls Derived destructor correctly (because virtual ~Base)
```

> *Always declare the destructor `virtual` in a polymorphic base class. Without it, `delete base_ptr` only calls the base destructor → resource leak in derived.*

---

## **9. Pure Virtual Function & Abstract Class**

A **pure virtual function** has no body in the base class — it's a contract that every concrete derived class *must* implement.

```cpp
virtual void draw() = 0;   // "= 0" makes it pure virtual
```

A class with at least one pure virtual function becomes an **abstract class**:
- *Cannot be instantiated directly*
- Acts as an interface / contract
- Derived classes must implement all pure virtual functions to be instantiatable

```cpp
class Shape {               // abstract class
public:
    virtual double area() = 0;   // pure virtual — no implementation
    virtual void draw()   = 0;
    virtual ~Shape() {}
};

class Circle : public Shape {
    double r;
public:
    Circle(double r) : r(r) {}
    double area() override { return 3.14 * r * r; }
    void draw()   override { cout << "Drawing circle"; }
};

// Shape s;      // ERROR — cannot instantiate abstract class
Shape* s = new Circle(5);   // OK — pointer to abstract type
```

> *Pure virtual functions enforce a design contract: "any concrete Shape must be able to report its area and draw itself."*

---

## **10. Diamond Problem & Virtual Inheritance**

The **diamond problem** occurs in multiple inheritance when two parent classes share a common grandparent, causing ambiguity — the grandparent's data is included *twice*.

```
       Animal
      /      \
   Dog        Cat
      \      /
       DogCat        ← ambiguous: which Animal's data?
```

```cpp
class Animal { public: int age; };
class Dog  : public Animal {};
class Cat  : public Animal {};
class DogCat : public Dog, public Cat {};

DogCat dc;
dc.age = 5;   // ERROR — ambiguous: Dog::Animal::age or Cat::Animal::age?
```

**Fix: Virtual Inheritance**

```cpp
class Animal { public: int age; };
class Dog  : virtual public Animal {};   // virtual keyword
class Cat  : virtual public Animal {};
class DogCat : public Dog, public Cat {};

DogCat dc;
dc.age = 5;   // OK — only one shared Animal subobject
```

- `virtual` inheritance tells the compiler: *share one single instance of the base class*
- The most-derived class (`DogCat`) is responsible for constructing the virtual base (`Animal`)
- Adds slight overhead: an extra pointer per virtual base in the object layout

> *Virtual inheritance solves the diamond problem by guaranteeing only one copy of the grandparent subobject exists, regardless of how many paths lead to it.*

---

## **Quick Reference — Interview One-Liners**

| Topic | One-liner |
|---|---|
| Dangling pointer | Pointer to freed memory; fix by nulling after `delete` |
| Memory leak | `new` without `delete`; fix with smart pointers |
| Stack vs Heap | Stack: fast/auto/small. Heap: manual/large/flexible |
| Compilation pipeline | Preprocessor → Compiler → Assembler → Linker → Loader |
| Call by value | Copy made; original unchanged |
| Call by reference | Alias; original changed; can't be null |
| Compile-time polymorphism | Overloading; resolved at compile time (early binding) |
| Runtime polymorphism | Virtual functions; resolved at runtime via vtable (late binding) |
| Overloading vs overriding | Same class/diff params vs child redefines parent's virtual |
| Virtual function | Enables runtime dispatch via vtable; always `virtual ~Base()` |
| Abstract class | Has pure virtual (`= 0`); can't be instantiated; defines interface |
| Diamond problem | Duplicate grandparent in multiple inheritance; fix: `virtual` inheritance |
