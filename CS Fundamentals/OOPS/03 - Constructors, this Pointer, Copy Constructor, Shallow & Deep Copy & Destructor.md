# Part 1 — Constructors, `this` Pointer & Copy Semantics

## Constructor 

### What Is a Constructor?

- A *special method* invokes called the **constructor** runs *automatically* on that memory

When you write `Teacher t1;`, two things happen in one shot:
1. **Memory** is <u>allocated</u> for `t1` (see [[01 - Introduction, OOP & Pointers#Stack vs Heap — Two Ways to Create Objects|Stack vs Heap]] for memory details)
2. **constructor** runs *automatically*

A constructor is a special member function that:
- Has the **same name** as the class
- Has **no return type** (not even `void`)
- Runs **automatically** only once (per object) the moment an object is created. (used to **initialise** the object's data members)
- Memory allocation occust when constructor is called.

If you don't write any constructor, the compiler silently generates a do-nothing one for you. The moment you write *any* constructor yourself, the compiler stops generating that default.

#### 2 ways to write a constructor

1. *Member Initializer List* (Preferred & Modern)
```cpp 
Student(int a, double cgpa) : age(a), cgpaPtr(new double(cgpa)) {}
```

2. *Normal Constructor Body*
```cpp 
Student(int a, double cgpa) {  
	age = a;  
	cgpaPtr = new double(cgpa);  
}
```

Initializer lists are preferred because members are **initialized directly** instead of being created first and then assigned.

What happens:

1. `age` is initialized first (with an unspecified/default/garbage value).
2. `cgpaPtr` is initialized first (with an unspecified value).
3. Then assignments happen inside the constructor body.

#### Aggregate Initialization

An **aggregate** is a simple class or struct whose members can be initialized directly using `{}` without writing a constructor.

```cpp
class Student { or //Stuct(public only)
  public:
    int age;
    string name;
};

Student s{20, "Arghadeep"};
```

---

### Non-Parameterized (Default) Constructor

Use this when all objects should start with the same default values:

```cpp
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
    Teacher(string name, string department, string subject, double salary=1000) {  //here 100 set as default used if salary not passed as argument
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

**Common trap:** If you define only a parameterized constructor, then `Teacher t1;` (no arguments) will fail to compile — the compiler no longer generates a default constructor. *Either add a default constructor explicitly, or use default parameter values.*

Always declare as ==public==.

>**Note**: **Parameters** are *variables* in a **function** **definition** wj **arguments** are the actual values **passed** to those parameters during a function call.

---

### Constructor Overloading

You can have multiple constructors in the same class — the compiler picks the right one based on the arguments:

- example of [[05 - Polymorphism — Overloading, Overriding & Virtual Functions | Polymorphism]]

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

## `this` Pointer

Every time you call a member function, C++ passes a hidden pointer, `this` to that function *pointing* to the **object it was called on**.

When you write `t1.setName("Shraddha")`, C++ internally transforms it to `Teacher::setName(&t1, "Shraddha")`. Inside `setName`, `this == &t1`.

- `this` is of type `ClassName* const` — the pointer itself can't be changed
- `this->member` is how you access the object's own data from inside a function (in class definition)
- `*this` is the entire object itself (useful for returning self)

**`this` does NOT exist in [[07 - The static Keyword in C++ OOPs#Part 1 — Static Variables, Members & Functions|Static]] functions** — static functions belong to the class, not any object.

```cpp
#include <iostream>
#include <string>
using namespace std;

class Teacher {
public:
    string name, department;
    // Use case 1: Disambiguate when parameter name matches member name
    Teacher(string name, string department) {
        this->name = name;        // "this->name" = object's member
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

## Copy Constructor

### What Is a Copy Constructor?

A copy constructor creates a **new object as a copy of an existing one**. Its signature is always:
```cpp
Teacher(const Teacher& obj)
//passed by reference
```

It fires in three situations:
1. `Teacher t2(t1);` — explicit copy construction *default copy constructor of c++*
2. `Teacher t2 = t1;` — copy-initialization (looks like assignment but calls copy constructor *(same as above)*
3. *Passing* or returning an *object by value* in a function

- **Why `const`?** — The copy constructor should only **read** from the source object and copy its data (not compulsory).

- **Why `&` (reference)?** — CRITICAL. If you took by value instead of reference, the compiler would need to copy the argument to pass it... which would call the copy constructor... which would copy its argument... infinite recursion → *==stack overflow*==. Reference avoids the copy.

```cpp
class Teacher {
public:
    string name, department, subject;
    double salary;

    Teacher(string name, string department, string subject, double salary) {
        this->name = name; this->department = department;
        this->subject = subject; this->salary = salary;
    } //constructor

    // Custom copy constructor
    Teacher(const Teacher& original) {
        // Copy each member from the original object
        this->name       = original.name;
        this->department = original.department;
        this->subject    = original.subject;
        this->salary     = original.salary;
        cout << "Copy constructor called!\n";
    } // Custom copy constructor
```

 **Memory** in Copy Constructors:
 
> ┌──────────────────┐  ← Teacher t1 (starts here)
│  name  (string)  │
│  dept  (string)  │
│  salary(double)  │
└──────────────────┘
>
>┌──────────────────┐  ← Teacher t2 (entirely separate block)
│  name  (string)  │
│  dept  (string)  │
│  salary(double)  │
└──────────────────┘
>
>Memory is *duplicated* to a new block

---

## Shallow Copy vs Deep Copy

### Shallow Copy

The **default copy constructor** does a **shallow (member-wise) copy** — it copies each member's value literally. (using this->name = obj->name)

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
1. Modifying `*s2.cgpaPtr` changes `s1`'s data too — they're *shared*
2. When both are destroyed, both destructors try to `delete` the same address → **double free → crash** 

> - The pointer does not know it was freed.
> - The memory allocator knows whether a memory block is allocated or already  freed. causing double free to lead to undefined behaviour.
>  - deleting nullptr is safe in c++.

*Note*: ==new== always created ==pointer==


```cpp
class Student {
public:
    string name;
    double* cgpaPtr;
    Student(string name, double cgpa) {
        this->name = name;
        cgpaPtr = new double(cgpa);
    }
    // Shallow Copy: copies pointer address, not data
    Student(const Student& obj) {
        name = obj.name;
        cgpaPtr = obj.cgpaPtr;   // both objects share same heap block (memory) *new*
    }
    ~Student() {
        delete cgpaPtr;
 // Destruction: At scope exit automatically called
// both objects try to delete same memory → double free
//unfined behaviour
    }
};
int main() {
    Student s1("Rahul", 8.9);
    Student s2(s1);      // s1.cgpaPtr == s2.cgpaPtr
    *s2.cgpaPtr = 9.2;   // changes s1's CGPA too
}
```

---

### Deep Copy — The Correct Solution

>Use when dynamically allocated memory involved (**heap memory**)

In a deep copy, for every pointer member, **allocate brand-new memory** and copy the *value* (not the address) into it. Each object owns its own independent heap block.

```cpp
class Student {
public:
    string name;
    double* cgpaPtr;
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

## Destructor

### What Is a Destructor?

A destructor is the exact counterpart to a constructor — it runs automatically when an object's **lifetime ends** (goes out of scope, or `delete` is called on a heap object).

1. `~` tilde prefix
2. no return type
3. no parameters
4. can't be overloaded

> automatically called on **reaching out of scope**

```cpp
	~ClassName(){
	}   // tilde prefix, no return type, no parameters, cannot be overloaded
```

The **default** compiler-generated destructor only cleans up stack-allocated members. It does **not** `delete` any pointer members — those *heap blocks are YOUR responsibility.*

- Manual destructor used in very low level development
```cpp
student1.~Student(); // Manual destructor call
```

```cpp
class Student {
public:
    string name;
    double* cgpaPtr;   // heap pointer — we must clean this up ourselves
    ~Student() {
        delete cgpaPtr;   // free the heap memory — MANDATORY if we allocated it
        cout << "Destructor: " << name << " destroyed\n";
    }
};
int main() {
    {//new scope
        Student s1("Rahul Kumar", 8.9);
        s1.getInfo();
    }   // ← destructor called HERE automatically, the moment s1 goes out of scope
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

### Destructor & Memory Leak

>Why we **need to fre**e dynamically allocated memory?

>A memory leak occurs when heap **memory allocated** with `new` is **not fred** using `delete`
>makes the memory **unreachable** but still **occupied**
> memory '**wasted**'

- In a normal C++ program, when `main()` ends, destructors of local objects are called automatically.  
- If destructors correctly free heap memory, there is no memory leak.
- 

**Rule:** Memory leak = allocated heap memory that is no longer reachable and was never freed.

### Virtual Destructor

A virtual destructor ensures that when a *derived object* is *deleted* *through* a *base-class pointer*, *both* the derived and base *destructors* are *called*.

If the base destructor is NOT virtual, only the base destructor runs — the derived destructor is skipped → *resource leak*.

```cpp title:code
class BaseTeacher {
public:
    virtual ~BaseTeacher() {
        cout << "Base Destructor\n";
    }
};

class DerivedTeacher : public BaseTeacher {
public:
    ~DerivedTeacher() {
        cout << "Derived Destructor\n";
    }
};

Base* ptr = new DerivedTeacher(); //base class pointer pointing to a derived class object.
delete ptr;
```

``` text title=output
Derived Destructor
Base Destructor
```

Why `virtual`?

- Ensures the **derived destructor runs first**.
- Prevents resource leaks when deleting derived objects through base pointers.
### Without Virtual Destructor

``` text title=output
Base Destructor
```

Only `Base` destructor may run, causing resource leaks in `Derived`.

### Rule

```text
If a class is intended to be used as a base class, make its destructor virtual.
```


---

## Rule of Three — The Golden Rule

> **If your class defines any one of these three, you almost certainly need all three:**

| Special Member           | When It Runs                                                            |
| ------------------------ | ----------------------------------------------------------------------- |
| Destructor               | Object goes out of scope or `delete` is called                          |
| Copy Constructor         | `Student s2(s1)`, `Student s2 = s1`, pass/return by value               |
| Copy Assignment Operator | **Existing object is assigned** the value of another object (`s2 = s1`) |

**Why all three?** A class with a heap pointer needs:
- Destructor → to free the heap memory
- Copy Constructor → to prevent shallow copy on initialization
- Copy Assignment → to prevent shallow copy on reassignment

Missing any one of the three leads to either **double-free** or **memory leak**.

```cpp
// Copy Assignment Operator (the third member of the Rule of Three)
Student& operator=(const Student& other) {
    if (this == &other) return *this;   // s1 = s1 (self-assignment) guard
    // operator= is a specail function name (returns *this i.e. Student&)
    delete cgpaPtr;                     // free what we currently own
    cgpaPtr = new double;                   // deep copy — allocate new memory
    *cgpaPtr = *other.cgpaPtr;             // copy the VALUE (same two-step as copy constructor)
    name    = other.name;
    return *this;   // return *this to allow chaining: s1 = s2 = s3;
}
```

>Note: **\*this** is basically **class&** i.e. dereferencing a pointer to class gives object reference

Student& as we are returning reference to original to itself:
allows chaining `s1 = s2 = s3;` equivalent to `s1 = (s2 = s3);`, `(s2 = s3)` return `s2` itself by reference

---

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

`std::initializer_list<T>` lets a constructor accept a **variable number of values of the same type**, written with `{}` *only*. This is how `std::vector` and other containers support `{1, 2, 3, 4, 5}` construction:

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

**Important: `std::initializer_list` constructor takes priority over other constructors** when `{}` is used: (not to be confused with [[#2 ways to write a constructor|member initializer list constructor]])

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
