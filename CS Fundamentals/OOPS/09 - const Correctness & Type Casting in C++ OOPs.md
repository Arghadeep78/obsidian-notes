# Part 1 — `const` Correctness

## What Is `const` Correctness?

`const` is a promise to the compiler: "this thing will NOT be modified." The compiler then enforces that promise — if you accidentally try to modify it, you get a compile error rather than a silent bug.

> **`const` correctness** = the discipline of marking variables, pointers, parameters, and member functions with `const` everywhere they shouldn't be modified, letting the compiler enforce immutability.

**Why bother?**
1. Bugs caught at **compile time** instead of runtime
2. Enables calling functions on `const` objects
3. Makes code intent clear ("this function won't change anything")
4. Allows the compiler to make performance optimizations

---

## `const` Variables

```cpp
const int MAX = 100;   // MAX is immutable after initialization
MAX = 200;             // ❌ ERROR: assignment to read-only variable
```

---

## `const` with Pointers — The Four Combinations

This is one of the **most tested C++ topics** in interviews. See [[01 - Introduction, OOP & Pointers#`const` with Pointers — Quick Reference|pointer basics]] for context. There are four combinations:

```cpp
int x = 10, y = 20;

// 1. Non-const pointer to non-const int
int* ptr1 = &x;
*ptr1 = 5;    // ✅ can change the value
ptr1  = &y;   // ✅ can redirect the pointer
// Both the pointer AND the value can change

// 2. Non-const pointer to CONST int  (read as: "pointer to const int")
const int* ptr2 = &x;
*ptr2 = 5;    // ❌ ERROR: value is const — cannot modify what ptr2 points to
ptr2  = &y;   // ✅ can redirect the pointer — ptr2 itself is not const

// 3. CONST pointer to non-const int  (read as: "const pointer to int")
int* const ptr3 = &x;
*ptr3 = 5;    // ✅ can change the value
ptr3  = &y;   // ❌ ERROR: pointer is const — cannot redirect it

// 4. CONST pointer to CONST int
const int* const ptr4 = &x;
*ptr4 = 5;    // ❌ ERROR: value is const
ptr4  = &y;   // ❌ ERROR: pointer is const
// Nothing can be changed
```

**The trick to remember:** Read the declaration **right-to-left**:
- `const int* ptr` → "ptr is a pointer to const int" → value is locked
- `int* const ptr` → "ptr is a const pointer to int" → pointer is locked
- `const int* const ptr` → "ptr is a const pointer to const int" → both locked

---

## `const` Member Functions

A `const` member function is one that **promises not to modify any member variable** of the object. Append `const` after the parameter list:

```cpp
class Circle {
    double radius;
public:
    Circle(double r) : radius(r) {}

    double getRadius() const {    // CONST function — promises: "I won't modify radius"
        return radius;            // just reading — safe
    }

    double area() const {         // CONST function — only reads, doesn't write
        return 3.14159 * radius * radius;
    }

    void setRadius(double r) {    // NON-CONST function — modifies radius
        radius = r;
    }
};

int main() {
    const Circle c(5.0);   // const object — cannot be modified after creation

    c.getRadius();         // ✅ const function on const object — allowed
    c.area();              // ✅ const function on const object — allowed
    c.setRadius(3.0);      // ❌ ERROR: calling non-const function on const object
}
```

**Rule:** A `const` object can ONLY call `const` member functions. Always mark functions `const` if they don't modify state — it's good practice and allows them to work on const objects.

---

## `const` Reference Parameters — The Most Common Use

This is the preferred way to pass objects to functions — avoids copying AND prevents modification:

```cpp
// BAD: passes by value — copies the ENTIRE object (expensive for large objects)
void printInfo(Student s) { cout << s.name; }

// ALSO BAD: passes by reference — but now the function could modify the caller's object
void printInfo(Student& s) { s.name = "Hacked!"; }  // silent mutation!

// GOOD: passes by const reference — no copy made, AND modification is prevented
void printInfo(const Student& s) {
    cout << s.name;        // ✅ reading is fine
    // s.name = "X";       // ❌ ERROR: s is const — cannot modify
}
```

| Passing Style | Copy Made? | Can Modify Caller's Object? | Best For |
|---|---|---|---|
| `T val` | Yes | No (modifying copy doesn't affect caller) | Primitives, when you need a local copy |
| `T& ref` | No | Yes | Intentionally modifying the caller's object |
| `const T& ref` | No | No | Read-only access — **use this by default** |

---

## `const` Return Type

```cpp
class Matrix {
    int data[10][10];
public:
    const int& get(int r, int c) const {    // returns const reference
        return data[r][c];
    }
};

Matrix m;
int val = m.get(0, 0);    // ✅ read — fine
m.get(0, 0) = 5;          // ❌ ERROR: result is const — can't assign through it
```

---

# Part 2 — Type Casting in OOPs

## Why C++ Has Four Casts

C has one cast: `(Type)expr`. It's powerful but dangerous — it tries multiple conversions silently without telling you which one it did.

C++ introduced four **explicit, named casts** that each do ONE specific thing. This makes code readable, searchable (you can grep for `dynamic_cast`), and safer.

```cpp
// C-style cast — what exactly is happening here? truncation? reinterpretation?
int i = (int)3.9;   // unclear intent

// C++ cast — explicit, obvious
int j = static_cast<int>(3.9);   // clearly: truncate double to int
```

---

## 1. `static_cast` — The "Safe Compile-Time" Cast

Use for **well-defined, related type conversions** that the compiler can verify at compile time. No runtime overhead, no runtime check.

```cpp
// Numeric conversions — explicit truncation
double d = 3.14;
int i = static_cast<int>(d);   // 3 — intent is clear: truncate

// Upcasting (derived → base pointer) — always safe
Dog* dog = new Dog();
Animal* a = static_cast<Animal*>(dog);   // safe upcast

// Downcasting (base → derived pointer) — UNSAFE if wrong type
Animal* a2 = new Animal();
Dog* d2 = static_cast<Dog*>(a2);         // compiles fine! But a2 is not a Dog
d2->bark();                              // UNDEFINED BEHAVIOUR — you lied to the compiler
```

**Use `static_cast` when:** You KNOW the types are correct. For downcasting when you're certain (or use `dynamic_cast` when you're not).

---

## 2. `dynamic_cast` — The "Safe Runtime" Cast

Use for **polymorphic downcasting** where you're not 100% sure what the actual type is. It checks the actual runtime type and returns `nullptr` on failure (instead of undefined behaviour). See [[05 - Polymorphism — Overloading, Overriding & Virtual Functions#Virtual Functions — The Core of Runtime Polymorphism|virtual functions]] for inheritance details.

**Requirement:** The class hierarchy MUST have at least one `virtual` function (enables RTTI — Runtime Type Information).

```cpp
class Animal { public: virtual ~Animal() {} };   // virtual function needed for dynamic_cast
class Dog    : public Animal { public: void bark() { cout << "Woof\n"; } };
class Cat    : public Animal { public: void meow() { cout << "Meow\n"; } };

int main() {
    Animal* a = new Dog();   // base pointer to actual Dog object

    // Try to downcast to Dog — this is correct
    Dog* d = dynamic_cast<Dog*>(a);
    if (d != nullptr) {
        d->bark();    // ✅ "Woof" — confirmed it's actually a Dog at runtime
    }

    // Try to downcast to Cat — this is wrong
    Cat* c = dynamic_cast<Cat*>(a);
    if (c == nullptr) {
        cout << "Not a Cat!\n";   // ✅ safe failure — no crash, no UB
    }

    delete a;
}
```

**`dynamic_cast` with references:** Instead of returning `nullptr` on failure (no such thing as null reference), it **throws `std::bad_cast`**:
```cpp
try {
    Cat& c = dynamic_cast<Cat&>(*a);   // throws std::bad_cast if *a is not a Cat
} catch (bad_cast& e) {
    cout << "Not a Cat!\n";
}
```

---

## `static_cast` vs `dynamic_cast` — The Key Difference

```
static_cast  = "Trust me, I know what I'm doing" — no runtime check
               → fast, but UB if you're wrong

dynamic_cast = "Let me verify at runtime"          — checks vtable
               → small overhead, but safe failure (nullptr or exception)
```

| When to use | `static_cast` | `dynamic_cast` |
|---|---|---|
| You're certain of the type | ✅ | Works too, but overkill |
| You're uncertain (user input, config) | ❌ Dangerous | ✅ Use this |
| Performance-critical inner loop | ✅ | Avoid if possible |
| Class has no virtual functions | ✅ | ❌ Won't compile (no RTTI) |

---

## 3. `const_cast` — Add or Remove `const`

The **only** cast that can add or remove `const` (or `volatile`) from a type. Use only for interfacing with old C APIs that don't accept `const` but genuinely don't modify the data.

```cpp
// Scenario: old C library function that takes char* (not const char*)
void legacyCPrint(char* str) { printf("%s", str); }  // old API, can't change it

const char* msg = "Hello";
legacyCPrint(const_cast<char*>(msg));  // strip const to match old API signature
// SAFE here because legacyCPrint doesn't actually modify the string

// DANGEROUS — actually modifying through const_cast = undefined behaviour
const int x = 10;
int* p = const_cast<int*>(&x);
*p = 20;   // ❌ UNDEFINED BEHAVIOUR — x was declared const, modifying it is UB

// SAFE — the original object is non-const; const was only added for the function interface
int y = 10;
const int* cp = &y;           // y is non-const, but we're passing it through a const interface
int* mp = const_cast<int*>(cp);
*mp = 20;                     // ✅ DEFINED BEHAVIOUR — y itself was never const
```

**The real rule — it's about the original object, not the pointer:**
- If the original variable was declared `const` → modifying it through `const_cast` is **undefined behaviour**
- If the original variable was declared non-const → stripping `const` from a pointer to it and modifying it is **perfectly safe and defined**

`const_cast` is a tool for interfacing with APIs that don't declare `const` correctly, not for breaking immutability. Never use it on data that was genuinely declared `const`.

---

## 4. `reinterpret_cast` — Raw Bit Reinterpretation

The most dangerous cast — no type safety whatsoever. It reinterprets the raw bits of one type as another type. Produces platform-dependent results.

```cpp
// Treating an integer as a pointer (low-level hardware code)
// Use intptr_t (from <cstdint>) — guaranteed wide enough to hold a pointer on any platform
// Using 'long' is wrong on 64-bit Windows (LLP64 model: long is 32-bit, pointers are 64-bit)
#include <cstdint>
intptr_t addr = 0xDEADBEEF;
int* ptr = reinterpret_cast<int*>(addr);   // treat that address as an int*

// Serialization: view a struct's raw bytes
struct Packet { int id; float value; };
Packet p = {42, 3.14f};
unsigned char* bytes = reinterpret_cast<unsigned char*>(&p);
// bytes[0..7] = raw memory of the struct — platform-dependent byte order!
```

**Use cases:** Embedded systems, OS/driver code, network packet parsing, memory-mapped I/O. **Never** use in normal application-level OOPs code.

---

## Cast Comparison Table

| Cast | Type Safety | Runtime Check | Use Case |
|---|---|---|---|
| `static_cast` | Compile-time check | No | Numeric conversions, upcasting, known-safe downcasts |
| `dynamic_cast` | Runtime check via vtable | Yes | Safe polymorphic downcasting (uncertain type) |
| `const_cast` | No type change | No | Add/remove `const` qualifier only |
| `reinterpret_cast` | None | No | Low-level bit reinterpretation |
| C-style `(Type)` | Tries all of the above | Depends | Avoid — too permissive, hides intent |

---

## RTTI — Runtime Type Information

`dynamic_cast` and `typeid` work because C++ tracks actual object types at runtime through a mechanism called **RTTI**.

```cpp
#include <typeinfo>

Animal* a = new Dog();
cout << typeid(*a).name() << "\n";    // prints compiler-specific name for Dog
cout << typeid(int).name() << "\n";   // "i" or "int" depending on compiler

// Check exact type at runtime
if (typeid(*a) == typeid(Dog)) {
    cout << "It's a Dog\n";
}
```

RTTI requires at least one `virtual` function in the class. It can be disabled with compiler flags (`-fno-rtti`) for performance-critical embedded code.

---

## Why Avoid C-Style Casts?

```cpp
double d = 3.9;

// C-style — ambiguous: is this a numeric conversion? a const removal? a reinterpretation?
int i = (int)d;

// Modern — explicit intent, compiler-checked, grep-able
int j = static_cast<int>(d);   // "I want to truncate this double"
```

C-style casts silently try a sequence of interpretations — `const_cast`, `static_cast` (sometimes combined with `const_cast`), and finally `reinterpret_cast` — using whichever succeeds first. This can hide very dangerous conversions.

Modern casts:
1. Express intent clearly in the code itself
2. Can be searched across a codebase (`grep dynamic_cast`)
3. Only do what they're designed for — if it's wrong, it fails loudly

---

## Interview Q&A

**Q: "What is `const` correctness?"**
A: Marking all variables, parameters, and member functions `const` wherever modification shouldn't happen. The compiler enforces immutability and allows those things to be used with const objects.

**Q: "Four pointer-const combinations?"**
A: `int*` (both mutable), `const int*` (value immutable, pointer movable), `int* const` (pointer fixed, value mutable), `const int* const` (both immutable). Read right-to-left.

**Q: "What is a `const` member function?"**
A: A member function declared with `const` after its parameter list, promising it won't modify any member variables. Only `const` member functions can be called on `const` objects.

**Q: "Four C++ casts?"**
A: `static_cast` (compile-time, related types), `dynamic_cast` (runtime check, polymorphic safe downcast), `const_cast` (add/remove const), `reinterpret_cast` (raw bit reinterpretation).

**Q: "When should you use `dynamic_cast`?"**
A: When you have a base class pointer and need to safely downcast to a derived class but aren't certain of the actual type. The base must have at least one virtual function. Returns `nullptr` on pointer failure.

**Q: "Why avoid C-style casts?"**
A: C-style casts silently try a sequence of interpretations (`const_cast`, `static_cast`, `reinterpret_cast`) — whichever compiles first succeeds. This makes them too permissive, hides dangerous conversions, and obscures intent. Modern casts are explicit and safer.

**Q: "What is RTTI?"**
A: Runtime Type Information — the mechanism that lets `dynamic_cast` and `typeid` check actual object types at runtime. Requires at least one virtual function in the class hierarchy.

---

## Special Topic — Type Conversion Operators

### What Is a Conversion Operator?

A **conversion operator** lets an object of your class be implicitly (or explicitly) converted to another type. It's the flip side of a converting constructor.

- **Converting constructor:** `MyClass(int x)` — converts `int` → `MyClass`
- **Conversion operator:** `operator int()` — converts `MyClass` → `int`

```cpp
class Temperature {
    double celsius;
public:
    Temperature(double c) : celsius(c) {}

    // Conversion operator: Temperature → double
    operator double() const {
        return celsius;
    }
};

int main() {
    Temperature t(36.6);
    double d = t;            // implicit conversion — calls operator double()
    cout << d << "\n";       // 36.6

    cout << t + 1.0 << "\n"; // 37.6 — t implicitly converted to double for arithmetic
}
```

**Syntax:**
```cpp
operator TargetType() const { return /* value of TargetType */; }
// No return type written before 'operator' — the target type IS the return type
```

---

### `explicit` Conversion Operator — Prevent Implicit Conversion

Just like constructors, conversion operators can be marked `explicit` to require an explicit cast:

```cpp
class SafeInt {
    int value;
public:
    SafeInt(int v) : value(v) {}

    explicit operator int() const { return value; }   // explicit — no silent conversion
    explicit operator bool() const { return value != 0; }
};

int main() {
    SafeInt s(42);

    int x = s;                      // ❌ COMPILE ERROR: conversion is explicit
    int y = static_cast<int>(s);    // ✅ explicit cast required
    int z = (int)s;                 // ✅ C-style cast also works

    if (s) { }                      // ✅ in boolean context (if/while/!/&&/||), explicit operator bool() fires automatically
    // This is called "contextual conversion to bool" — the only context where an explicit conversion operator is invoked implicitly.
    // That's why operator bool() is almost always marked explicit: it prevents obj+5 from silently treating obj as 0 or 1,
    // while still allowing if(obj) to work naturally.
}
```

**Why `explicit operator bool()` is the standard pattern:** Without `explicit`, an object with `operator bool()` can accidentally be used in arithmetic (`obj + 5` would convert `obj` to `0` or `1`). With `explicit`, it only fires in boolean contexts (`if`, `while`, `!`, `&&`, `||`) — the compiler makes a special exception for these.

---

### Conversion Operator vs Constructor — Summary

| Direction | Mechanism | Keyword |
|---|---|---|
| `int` → `MyClass` | Converting constructor: `MyClass(int)` | `explicit` to disable implicit |
| `MyClass` → `int` | Conversion operator: `operator int()` | `explicit` to disable implicit |

```cpp
class Celsius {
    double temp;
public:
    Celsius(double t) : temp(t) {}            // int/double → Celsius
    explicit operator double() const { return temp; }  // Celsius → double (explicit)
    explicit operator int() const { return (int)temp; } // Celsius → int (explicit)
};

Celsius c(36.6);
double d = static_cast<double>(c);  // 36.6
int    i = static_cast<int>(c);     // 36
```

**Interview Answer:** "A conversion operator defines how an object of your class converts to another type. Declared as `operator TargetType() const`. Mark it `explicit` to prevent unintended implicit conversions — especially for `operator bool()`, which would otherwise let objects participate in arithmetic."
