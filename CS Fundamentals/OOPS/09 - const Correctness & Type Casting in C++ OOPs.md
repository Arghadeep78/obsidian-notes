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
// p1 is a pointer to an int
int* p1 = &x;             // Pointer: p1 = &y;  ✔ | Value: *p1 = 30;  ✔
// p2 is a pointer to a const int
const int* p2 = &x;       // Pointer: p2 = &y;  ✔ | Value: *p2 = 30;  ❌
// p3 is a const pointer to an int
int* const p3 = &x;       // Pointer: p3 = &y;  ❌ | Value: *p3 = 30;  ✔
// p4 is a const pointer to a const int
const int* const p4 = &x; // Pointer: p4 = &y;  ❌ | Value: *p4 = 30;  ❌
```

**The trick to remember:** Read the declaration **right-to-left**:
- `const int* ptr` → "ptr is a pointer to const int" → value is locked
- `int* const ptr` → "ptr is a const pointer to int" → pointer is locked
- `const int* const ptr` → "ptr is a const pointer to const int" → both locked

---
## `const` Member Functions

A member function marked with `const` promises **not to modify the object's data members**.

```cpp
class Circle {
public:
    double area() const;   // const function
    void setRadius();      // non-const function
};
```
#### Rule
- const object → can call ONLY const member functions (have to declare even if does not modify)
- non-const object → can call both const and non-const functions
```cpp
const Circle c(5);
c.area();       // OK
c.setRadius();  // ERROR
```
#### Good Practice
- If a function only reads data and does not modify the object,mark it const.
Benefits:
- Prevents accidental modification.
- Allows calls on const objects.
- Improves code correctness.
---

## `const` Reference Parameters — The Most Common Use

This is the preferred way to pass objects to functions — avoids copying AND prevents modification:

```cpp
// Bad: pass by value
// Copies entire object (expensive)
void printInfo(Student s);
// Bad: pass by reference
// No copy, but can modify original object
void printInfo(Student& s);
// Good: pass by const reference
// No copy + cannot modify original object
void printInfo(const Student& s);
```
---

## `const` Return Type
- const return type *prevents modification* of the *returned* value/reference.  
- Most useful when returning a reference or pointer to internal data.

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
C-style cast `(Type)expr` is powerful but unsafe because it may perform different conversions silently.
```cpp
int i = (int)3.9;                // unclear
int j = static_cast<int>(3.9);   // explicit truncation
```
---
## 1. `static_cast` — Compile-Time Cast
For safe, related type conversions checked at compile time.
```cpp
double d = 3.14;
int i = static_cast<int>(d);     // 3
Dog* dog = new Dog();
Animal* a = static_cast<Animal*>(dog);   // safe upcast (Upcast)
Animal* a2 = new Animal();
Dog* d2 = static_cast<Dog*>(a2);         // compiles, but unsafe(Down Cast)
```
### Use When
- Numeric conversions
- Upcasting
- Downcasting only when you're certain of the actual type
- *Trust me, I know the type.*
## 2. `dynamic_cast` — Runtime-Checked Cast
For safe downcasting in polymorphic hierarchies.
**Requires at least one `virtual` function.**

```cpp
Animal* a = new Dog();
Dog* d = dynamic_cast<Dog*>(a);   // success
Cat* c = dynamic_cast<Cat*>(a);   // nullptr
```
Reference version throws `std::bad_cast`.
```cpp
Cat& c = dynamic_cast<Cat&>(*a);   // throws on failure
```
### Use When
- Actual type is uncertain
- Safe polymorphic downcasting
- *Verify the type at runtime.*

|                         | static_cast | dynamic_cast         |
| ----------------------- | ----------- | -------------------- |
| Runtime check           | n           | y                    |
| Speed                   | fast        | slight overhead      |
| Wrong cast              | undefined   | nullptr` / exception |
| Virtual function needed | y           | n                    |

## 3. `const_cast` — Add/Remove `const`

Only cast that can change `const`/`volatile` qualifiers.
```cpp
const char* msg = "Hello";  
legacyPrint(const_cast<char*>(msg));
```
### Safe (Modifying a non-const object via a const reference)

```cpp
int x = 10; // Original object is NOT const
const int& ref = x; 
// Strip const to modify
const_cast<int&>(ref) = 20; // Safe: x is now 20
```
### Undefined Behaviour (Modifying a genuinely const object)

```cpp
const int x = 10; // Original object IS const
const int* ptr = &x;
// Strip const and attempt modification
int* modifiablePtr = const_cast<int*>(ptr);
*modifiablePtr = 20; // UNDEFINED BEHAVIOR: May crash or optimize unexpectedly
```

## 4. `reinterpret_cast` — Bit Reinterpretation

Converts pointer type to other pointer type
OR pointer to an integer (and vice versa), at the raw bit level.

```cpp
float f = 3.14f;

// Compile Error: Compiler knows a float pointer isn't an int pointer
// int* ip = static_cast<int*>(&f); 

// Compiles: Forces the bits of the float to be read as an integer
int* ip = reinterpret_cast<int*>(&f); 

std::cout << *ip; // Outputs a massive integer (the raw IEEE 754 bits of 3.14)
```
#### Use Cases
Low-level memory reinterpretation
- OS/driver code
- Embedded systems
- Memory-mapped I/O
- Serialization
## RTTI (Runtime Type Information)

Mechanism Used by `dynamic_cast` and `typeid`  to check actual object types at runtime.

```cpp
Animal* a = new Dog();
cout << typeid(*a).name();
if (typeid(*a) == typeid(Dog))
    cout << "Dog";
//P6Animal (a pointer to a class named "Animal" of length 6
//Dog
```
- Requires atleast one virtual function

## Why Avoid C-Style Casts?

```cpp
int i = (int)d;                  // ambiguous
int j = static_cast<int>(d);     // explicit
```
C-style casts may internally perform:
1. `const_cast`
2. `static_cast`
3. `reinterpret_cast`
whichever succeeds first.

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
A conversion operator allows an object of a class to be converted to another type (the opposite of a constructor).
- **Constructor:** `TypeA` → `MyClass`
- **Conversion Operator:** `MyClass` → `TypeB`
### Syntax & Example
```cpp
class Temperature {
    double celsius;
public:
    Temperature(double c) : celsius(c) {}
    // No return type specified; the target type IS the return type
    operator int() const { return floor(celsius)+1; } 
};
Temperature t(36.6);
int d = t; // Implicit conversion: ✅ d becomes 37 (36.0 + 1)
```
### The `explicit` Keyword
Marking an operator `explicit` prevents accidental implicit conversions and requires a `static_cast`.
```cpp
explicit operator int() const { return value; }

int x = t;                  // ❌ Compile Error
int x = static_cast<int>(t); // ✅ Allowed
```
##### The `operator bool()` Exception
- **The Exception:** The compiler permits **contextual conversion**, meaning `explicit operator bool()` still triggers automatically inside conditional contexts (`if`, `while`, `!`, `&&`, `||`).