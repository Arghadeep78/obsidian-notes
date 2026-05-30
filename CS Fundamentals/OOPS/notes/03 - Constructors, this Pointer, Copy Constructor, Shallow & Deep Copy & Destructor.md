# Part 1 — Constructors, `this` Pointer & Copy Semantics

## Constructor — The "Setup Man"

### What Is a Constructor?

When you write `Teacher t1;`, two things happen in one shot:
1. Memory is allocated for `t1` (see [[01 - Introduction, OOP & Pointers#Stack vs Heap — Two Ways to Create Objects|Stack vs Heap]] for memory details)
2. A special function called the **constructor** runs automatically on that memory

A constructor is a special member function that:
- Has the **same name** as the class
- Has **no return type** (not even `void`)
- Runs **automatically** the moment an object is created
- Is used to **initialise** the object's data members to sensible defaults

If you don't write any constructor, the compiler silently generates a do-nothing one for you. The moment you write *any* constructor yourself, the compiler stops generating that default.

---

### Non-Parameterized (Default) Constructor

Use this when all objects should start with the same default values:

```cpp
#include <iostream>
#include <string>
using namespace std;

class Teacher {
public:
    string name, department, subject;
    double salary;

    // Constructor — same name as class, no return type
    Teacher() {
        // This runs automatically when any Teacher object is created
        department = "Computer Science";   // set a default value
        salary = 0;
        cout << "Constructor called!\n";
    }
};

int main() {
    Teacher t1;   // prints: Constructor called! — automatic, you didn't call it
    Teacher t2;   // prints: Constructor called! — one call per object
    cout << t1.department << "\n";   // Computer Science — set by constructor
    cout << t2.department << "\n";   // Computer Science — t2 also got it
}
```

---

### Parameterized Constructor

Use this when each object needs different initial values. Pass them at creation time:

```cpp
class Teacher {
public:
    string name, department, subject;
    double salary;

    // Parameterized constructor — takes arguments at object creation
    Teacher(string name, string department, string subject, double salary) {
        // PROBLEM: parameter names clash with member names!
        // "name = name;" would be assigning the parameter to itself — a bug!
        // FIX: use "this->" to explicitly refer to the OBJECT's member
        this->name       = name;          // object's name = parameter name
        this->department = department;
        this->subject    = subject;
        this->salary     = salary;
    }

    void getInfo() {
        cout << name << " | " << department << " | " << subject
             << " | " << salary << "\n";
    }
};

int main() {
    // Values passed right at creation — object is fully set up instantly
    Teacher t1("Shraddha", "Computer Science", "C++", 25000);
    t1.getInfo();   // Shraddha | Computer Science | C++ | 25000
}
```

**Common trap:** If you define only a parameterized constructor, then `Teacher t1;` (no arguments) will fail to compile — the compiler no longer generates a default constructor. Either add a default constructor explicitly, or use default parameter values.

---

### Constructor Overloading

You can have multiple constructors in the same class — the compiler picks the right one based on the arguments:

```cpp
class Teacher {
public:
    string name, department;

    Teacher() {                        // no-arg version
        department = "Computer Science";
        cout << "Default constructor\n";
    }

    Teacher(string n, string d) {      // full version
        name = n;
        department = d;
        cout << "Parameterized constructor\n";
    }

    Teacher(const Teacher& obj) {      // copy version (covered later)
        name = obj.name;
        department = obj.department;
        cout << "Copy constructor\n";
    }
};

int main() {
    Teacher t1;                   // calls the no-arg version
    Teacher t2("Shraddha", "CS"); // calls the full version
    Teacher t3(t2);               // calls the copy version
}
```

**Classic interview trap:** `Teacher t1();` looks like it creates an object, but it actually **declares a function** named `t1` that returns a `Teacher`. This is called the **Most Vexing Parse**. Always write `Teacher t1;` (no parentheses) for the default constructor.

---

## `this` Pointer — The Hidden "I Am" Pointer

### What Is `this`?

Every time you call a member function, C++ passes a hidden pointer to that function pointing to the **object it was called on**. This hidden pointer is called `this`.

When you write `t1.setName("Shraddha")`, C++ internally transforms it to `Teacher::setName(&t1, "Shraddha")`. Inside `setName`, `this == &t1`.

- `this` is of type `ClassName* const` — the pointer itself can't be changed
- `this->member` is how you access the object's own data from inside a function
- `*this` is the entire object itself (useful for returning self)

**`this` does NOT exist in static functions** — static functions belong to the class, not any object.

```cpp
#include <iostream>
#include <string>
using namespace std;

class Teacher {
public:
    string name, department;

    // Use case 1: Disambiguate when parameter name matches member name
    Teacher(string name, string department) {
        this->name       = name;        // "this->name" = object's member
        this->department = department;  // "name" alone = the parameter
    }

    // Use case 2: Method chaining — return the object itself so calls can be chained
    Teacher& setName(string name) {
        this->name = name;
        return *this;   // return the whole object by reference
    }

    Teacher& setDept(string dept) {
        this->department = dept;
        return *this;
    }

    void print() { cout << name << " | " << department << "\n"; }
};

int main() {
    Teacher t1("Shraddha", "CS");
    t1.print();

    // Method chaining: each setter returns *this, so you can chain calls
    Teacher t2("", "");
    t2.setName("Rahul").setDept("EE").print();
    //         ↑ returns t2     ↑ called on t2    ↑ called on t2
}
```

---

## Copy Constructor — Making a Clone

### What Is a Copy Constructor?

A copy constructor creates a **new object as a copy of an existing one**. Its signature is always:
```cpp
ClassName(const ClassName& obj)
```

It fires in three situations:
1. `Teacher t2(t1);` — explicit copy construction
2. `Teacher t2 = t1;` — copy-initialization (looks like assignment but calls copy constructor)
3. Passing or returning an object by value in a function

**Why `const`?** — The original shouldn't be modified while being copied.
**Why `&` (reference)?** — CRITICAL. If you took by value instead of reference, the compiler would need to copy the argument to pass it... which would call the copy constructor... which would copy its argument... **infinite recursion → stack overflow**. Reference avoids the copy.

```cpp
class Teacher {
public:
    string name, department, subject;
    double salary;

    Teacher(string name, string department, string subject, double salary) {
        this->name = name; this->department = department;
        this->subject = subject; this->salary = salary;
    }

    // Custom copy constructor
    Teacher(const Teacher& original) {
        // Copy each member from the original object
        this->name       = original.name;
        this->department = original.department;
        this->subject    = original.subject;
        this->salary     = original.salary;
        cout << "Copy constructor called!\n";
    }

    void getInfo() {
        cout << name << " | " << department << " | " << subject << " | " << salary << "\n";
    }
};

int main() {
    Teacher t1("Shraddha", "Computer Science", "C++", 25000);

    Teacher t2(t1);    // explicit: calls copy constructor
    Teacher t3 = t1;   // copy-initialization: ALSO calls copy constructor (not assignment!)

    t2.getInfo();   // Shraddha | Computer Science | C++ | 25000
    t3.getInfo();   // same — independent copy
}
```

---

## Shallow Copy vs Deep Copy — The Most Common Interview Topic

### Shallow Copy — Why It Breaks

The **default compiler-generated copy constructor** does a **shallow (memberwise) copy** — it copies each member's value literally. See [[01 - Introduction, OOP & Pointers#Pointers to Objects — Dot vs Arrow|pointers]] for memory reference details.

For simple types (`int`, `double`, `string`) this is fine. For **pointer members**, this is dangerous:

```
Object s1:  name = "Rahul"    cgpaPtr → [address 5000] → value 8.9
                                                ↑
Shallow copy just copies the address (5000), not the value
                                                ↓
Object s2:  name = "Rahul"    cgpaPtr → [address 5000] → value 8.9
                                          SAME address!
```

Both `s1` and `s2` point to the **exact same memory location**. This causes two problems:
1. Modifying `*s2.cgpaPtr` changes `s1`'s data too — they're shared
2. When both are destroyed, both destructors try to `delete` the same address → **double free → crash**

```cpp
class Student {
public:
    string name;
    double* cgpaPtr;   // pointer to heap memory — the danger zone

    Student(string name, double cgpa) {
        this->name = name;
        cgpaPtr    = new double;   // allocate heap memory
        *cgpaPtr   = cgpa;
    }

    // SHALLOW copy (what the compiler generates by default — DO NOT USE with pointers)
    Student(const Student& obj) {
        name    = obj.name;
        cgpaPtr = obj.cgpaPtr;   // copies the ADDRESS, not the value!
                                  // now both objects point to the same heap block
    }

    void getInfo() {
        cout << name << " | CGPA: " << *cgpaPtr << "\n";
    }

    ~Student() {
        delete cgpaPtr;   // both s1 and s2 have cgpaPtr pointing to the SAME address
                          // s1's destructor runs first and frees it
                          // s2's destructor then tries to delete the same address → DOUBLE FREE → crash
    }
};

int main() {
    Student s1("Rahul Kumar", 8.9);
    Student s2(s1);          // shallow copy — s2.cgpaPtr == s1.cgpaPtr (same address!)

    *s2.cgpaPtr = 9.2;       // "I'm only changing s2's CGPA..."
    s1.getInfo();            // BUG: prints 9.2 — s1 got corrupted because they share memory!
    // At scope exit: s2's destructor deletes cgpaPtr, then s1's destructor also
    // tries to delete the same address → DOUBLE FREE → undefined behaviour / crash
}
```

---

### Deep Copy — The Correct Solution

In a deep copy, for every pointer member, **allocate brand-new memory** and copy the *value* (not the address) into it. Each object owns its own independent heap block.

```cpp
class Student {
public:
    string name;
    double* cgpaPtr;

    Student(string name, double cgpa) {
        this->name = name;
        cgpaPtr    = new double;
        *cgpaPtr   = cgpa;
    }

    // DEEP copy constructor — correct way
    Student(const Student& obj) {
        name    = obj.name;
        cgpaPtr = new double;         // NEW allocation — brand new memory block
        *cgpaPtr = *obj.cgpaPtr;      // copy the VALUE at that address, not the address itself
        cout << "Deep copy constructor called!\n";
    }

    void getInfo() {
        cout << name << " | CGPA: " << *cgpaPtr << "\n";
    }

    ~Student() {
        delete cgpaPtr;   // each object frees its own block — no conflict
    }
};

int main() {
    Student s1("Rahul Kumar", 8.9);
    Student s2(s1);          // deep copy — s2 gets its own separate heap block

    s2.name    = "Neha Kumar";
    *s2.cgpaPtr = 9.2;       // only s2's memory changes — s1 is untouched

    s1.getInfo();   // Rahul Kumar | CGPA: 8.9  ← unchanged ✅
    s2.getInfo();   // Neha Kumar  | CGPA: 9.2
}
```

**Memory layout after deep copy:**
```
s1:  name="Rahul"   cgpaPtr → [address 5000] → 8.9   (s1's own block)
s2:  name="Neha"    cgpaPtr → [address 6000] → 9.2   (s2's own NEW block)
```
Completely independent. No sharing.

---

# Part 2 — Destructor, Rule of Three & Special Topics

## Destructor — The "Cleanup Man"

### What Is a Destructor?

A destructor is the exact counterpart to a constructor — it runs automatically when an object's **lifetime ends** (goes out of scope, or `delete` is called on a heap object).

```cpp
~ClassName()   // tilde prefix, no return type, no parameters, cannot be overloaded
```

The **default** compiler-generated destructor only cleans up stack-allocated members. It does **not** `delete` any pointer members — those heap blocks are YOUR responsibility.

```cpp
#include <iostream>
#include <string>
using namespace std;

class Student {
public:
    string name;
    double* cgpaPtr;   // heap pointer — we must clean this up ourselves

    Student(string name, double cgpa) {
        this->name = name;
        cgpaPtr    = new double;
        *cgpaPtr   = cgpa;
        cout << "Constructor: " << name << " created\n";
    }

    void getInfo() {
        cout << name << " | CGPA: " << *cgpaPtr << "\n";
    }

    ~Student() {
        delete cgpaPtr;   // free the heap memory — MANDATORY if we allocated it
        cout << "Destructor: " << name << " destroyed\n";
    }
};

int main() {
    {
        Student s1("Rahul Kumar", 8.9);
        s1.getInfo();
    }   // ← destructor fires HERE automatically, the moment s1 goes out of scope
        //   NOT at the end of main() — at the end of the BLOCK

    cout << "After the block ends\n";
    return 0;
}

// Output:
// Constructor: Rahul Kumar created
// Rahul Kumar | CGPA: 8.9
// Destructor: Rahul Kumar destroyed   ← fires at end of { } block
// After the block ends
```

### Why `virtual` Destructor in Base Class?

This is one of the **most asked SDE-1 interview questions**. If you delete a derived class object through a base class pointer and the base destructor is NOT virtual, only the base destructor runs — the derived destructor is skipped → resource leak.

```cpp
class Base  { public: ~Base()    { cout << "Base dtor\n"; } };   // NOT virtual!
class Child : public Base { public: ~Child() { cout << "Child dtor\n"; } };

Base* ptr = new Child();
delete ptr;
// Only prints: "Base dtor" — Child's destructor is SKIPPED → memory leak!

// FIX: make base destructor virtual
class Base { public: virtual ~Base() { cout << "Base dtor\n"; } };
// Now delete ptr; prints: "Child dtor" then "Base dtor" — correct!
```

---

## Stack vs Heap Memory — Quick Visual

```
Stack (automatic, fast):         Heap (manual, flexible):
┌────────────────────┐           ┌─────────────────────────────┐
│  local variables   │           │  new double → address 5000  │
│  objects (no new)  │           │  new int[10] → address 6000 │
│  freed at scope end│           │  persists until delete'd     │
└────────────────────┘           └─────────────────────────────┘
```

- **Forget `delete` on heap** → memory leak (block is reserved but never freed)
- **After `delete ptr;`** → `ptr` is a dangling pointer. Always set `ptr = nullptr;` after deleting
- **`delete` vs `delete[]`** → use `delete[]` for arrays (`new int[10]`), `delete` for single objects

---

## Rule of Three — The Golden Rule

> **If your class defines any one of these three, you almost certainly need all three:**

| Special Member | When It Runs |
|---|---|
| Destructor | Object goes out of scope or `delete` is called |
| Copy Constructor | `Student s2(s1)`, `Student s2 = s1`, pass/return by value |
| Copy Assignment Operator | `s2 = s1` when s2 **already exists** |

**Why all three?** A class with a heap pointer needs:
- Destructor → to free the heap memory
- Copy Constructor → to prevent shallow copy on initialization
- Copy Assignment → to prevent shallow copy on reassignment

Missing any one of the three leads to either **double-free** or **memory leak**.

```cpp
// Copy Assignment Operator (the third member of the Rule of Three)
Student& operator=(const Student& other) {
    if (this == &other) return *this;   // guard against s1 = s1 (self-assignment)

    delete cgpaPtr;                     // free what we currently own
    cgpaPtr = new double;                   // deep copy — allocate new memory
    *cgpaPtr = *other.cgpaPtr;             // copy the VALUE (same two-step as copy constructor)
    name    = other.name;
    return *this;   // return *this to allow chaining: s1 = s2 = s3;
}
```

---

## Operator Overloading (Bonus)

> Redefine what built-in operators (`+`, `==`, `<<`, etc.) do for your custom types.

```cpp
class Vector2D {
public:
    double x, y;
    Vector2D(double x = 0, double y = 0) : x(x), y(y) {}

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
    }
};

int main() {
    Vector2D v1(1, 2), v2(3, 4);
    Vector2D v3 = v1 + v2;        // calls operator+
    cout << v3 << "\n";           // (4, 6) — calls operator<<
    cout << (v1 == v2) << "\n";   // 0 (false)
}
```

**Operators you CANNOT overload:** `::` (scope), `.` (member access), `.*`, `?:` (ternary), `sizeof`

---

## `explicit` Keyword — Prevent Surprise Conversions

> `explicit` on a single-argument constructor tells the compiler: "never use this constructor for *implicit* type conversions."

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

---

## Special Topic — `= default` and `= delete`

### The Problem Without These

Before C++11, you couldn't easily control whether the compiler generated special members. To disable copying you'd declare the copy constructor `private` — a hack. To request the default — you'd just not write one and hope the compiler cooperated.

C++11 added two clean keywords: `= default` and `= delete`.

### `= default` — Explicitly Request the Compiler-Generated Version

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

  Each special member has its own suppression rule — they are not all the same:
  - **Default constructor** — suppressed if you write *any* user-defined constructor (parameterized, copy, or move). `MyClass() = default;` explicitly re-requests it.
  - **Copy constructor / copy assignment** — suppressed if you declare a move constructor or move assignment operator. If you only declare a destructor (but no move operations), the copy operations are still generated by the compiler — but *relying on this implicit generation is deprecated practice* (the standard may remove it in a future revision, so define them explicitly whenever you have a user-declared destructor).
  - **Move constructor / move assignment** — suppressed if you declare *any* of: destructor, copy constructor, or copy assignment operator. This is the main reason the Rule of Five exists: declaring any one of them kills the generated move operations.

```cpp
class Teacher {
public:
    string name;
    Teacher(string n) : name(n) {}   // user-defined constructor — compiler stops generating default ctor

    Teacher() = default;             // explicitly bring back the default ctor
};

Teacher t1;             // ✅ works — default ctor restored via = default
Teacher t2("Shraddha"); // ✅ works — parameterized ctor
```

### `= delete` — Explicitly Disable a Function

```cpp
class NonCopyable {
public:
    NonCopyable() = default;

    NonCopyable(const NonCopyable&) = delete;            // copying is DISABLED
    NonCopyable& operator=(const NonCopyable&) = delete; // copy assignment DISABLED

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
    void process(double) = delete;   // prevent silent int → double conversion
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
Teacher t1{"Shraddha", "CS", "C++", 25000};   // calls matching constructor
```

**Bonus: `{}` prevents narrowing conversions** (a common source of silent bugs):
```cpp
int a = 3.14;    // old style: silently truncates to 3 — no warning
int b{3.14};     // ❌ COMPILE ERROR: narrowing conversion not allowed in {}
                 // {} is stricter — it protects you from accidental data loss
```

### `std::initializer_list` — Constructing from a List of Values

`std::initializer_list<T>` lets a constructor accept a **variable number of values of the same type**, written with `{}`. This is how `std::vector` and other containers support `{1, 2, 3, 4, 5}` construction:

```cpp
#include <initializer_list>
#include <iostream>
using namespace std;

class NumberSet {
    vector<int> data;
public:
    // Constructor taking initializer_list — called when you write NumberSet{1,2,3}
    NumberSet(initializer_list<int> values) {
        for (int v : values)
            data.push_back(v);
    }

    void print() {
        for (int v : data) cout << v << " ";
        cout << "\n";
    }
};

int main() {
    NumberSet s{10, 20, 30, 40, 50};   // calls the initializer_list constructor
    s.print();   // 10 20 30 40 50

    // This is exactly how vector works:
    vector<int> v{1, 2, 3, 4, 5};     // vector's initializer_list constructor
}
```

**Important: `initializer_list` constructor takes priority over other constructors** when `{}` is used:
```cpp
class Tricky {
public:
    Tricky(int n, int val) { cout << "Regular ctor: " << n << " copies of " << val << "\n"; }
    Tricky(initializer_list<int> lst) { cout << "initializer_list ctor, size=" << lst.size() << "\n"; }
};

Tricky a(3, 5);    // Regular ctor: 3 copies of 5   — () picks regular
Tricky b{3, 5};    // initializer_list ctor, size=2  — {} picks initializer_list!
```

This is a well-known C++11 gotcha: `vector<int> v(5, 0)` creates 5 zeros, but `vector<int> v{5, 0}` creates a 2-element vector `[5, 0]`.

---

## Special Topic — Aggregates & POD Types

### What Is an Aggregate?

An **aggregate** is a class/struct with no user-declared constructors, no private/protected non-static data members, no base classes, and no virtual functions. Aggregates support **aggregate initialization** — initializing directly from a brace list without needing a constructor:

```cpp
struct Point {        // aggregate: no constructor, all public
    int x;
    int y;
    int z;
};

Point p1 = {1, 2, 3};   // aggregate initialization — values assigned in order
Point p2 {4, 5, 6};     // same with uniform initialization syntax
Point p3 {1};            // partial init — x=1, y=0, z=0 (rest zero-initialized)

cout << p1.x << " " << p1.y << " " << p1.z << "\n";  // 1 2 3
```

### What Is a POD Type?

**POD (Plain Old Data)** = a type that is compatible with C. It's both an aggregate AND has no non-trivial special members (no user-defined constructors, destructors, or copy operators):

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
Packet p = {42, 3.14f, 'A'};
// Safe for binary ops:
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

## Rapid-Fire Interview Q&A

| Question | Answer |
|---|---|
| What is a constructor? | Special member function, same name as class, no return type, auto-called at object creation to initialise members. |
| Can a constructor be `virtual`? | No — vtable isn't set up until construction completes, so `virtual` constructor makes no sense. |
| `Teacher t1()` vs `Teacher t1;`? | `t1()` is a *function declaration* (Most Vexing Parse). `t1;` creates an object. |
| Why `const&` in copy constructor? | (1) Reference avoids infinite recursion from passing by value. (2) `const` prevents accidental mutation of the original. |
| Shallow vs deep copy? | Shallow copies the pointer's address (both objects share heap → double-free risk). Deep allocates new memory and copies the value (independent → safe). |
| Rule of Three? | If you define any of destructor, copy constructor, or copy assignment — define all three. Class with heap pointer needs all three. |
| Why `virtual` destructor in base? | Without it, `delete base_ptr` on a derived object skips the derived destructor → resource leak + undefined behaviour. |
| What is RAII? | Resource Acquisition Is Initialisation — acquire resources in constructor, release in destructor. Guarantees cleanup even on exceptions. |
| After `delete ptr`, what is `ptr`? | A dangling pointer — pointing to freed memory. Always set `ptr = nullptr` after deleting. |
| `delete` vs `delete[]`? | `delete` for a single object (`new T`). `delete[]` for arrays (`new T[n]`). Wrong pairing = undefined behaviour. |
