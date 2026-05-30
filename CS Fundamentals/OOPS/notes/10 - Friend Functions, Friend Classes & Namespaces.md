# Part 1 — `friend`: Controlled Access to Private Members

## What Is `friend` and Why Does It Exist?

OOPs says: hide your internals via [[02 - Class, Object, Access Specifiers & Encapsulation#Access Specifiers — The Core Rules|access specifiers]]. Private members stay private.

But sometimes, two classes are so tightly coupled that one genuinely *needs* direct access to the other's private data — and adding public getters/setters just to satisfy that one helper would pollute the public interface for everyone.

`friend` solves this: grant **targeted, explicit, controlled access** to one specific function or class — without opening that access to the whole world.

> A **`friend` function** (or `friend` class) declared inside a class gets access to that class's `private` and `protected` members, even though it's defined outside the class.

**Classic real-world need:** The `operator<<` overload for printing must sit outside the class (because `ostream` must be on the left-hand side of `<<`), but it needs to read private member data to print them. Solution: make it a `friend`.

---

## `friend` Function — The Basics

```cpp
#include <iostream>
using namespace std;

class BankAccount {
private:
    string owner;
    double balance;      // private — hidden from outside

public:
    BankAccount(string o, double b) : owner(o), balance(b) {}

    // DECLARATION: give auditAccount function permission to see our private data
    // This is a GRANT of access, not a function definition
    friend void auditAccount(const BankAccount& acc);
};

// DEFINITION: auditAccount is NOT a member of BankAccount
// But because of the friend declaration, it CAN access private owner and balance
void auditAccount(const BankAccount& acc) {
    cout << "Owner:   " << acc.owner   << "\n";   // ✅ private — allowed via friend
    cout << "Balance: " << acc.balance << "\n";   // ✅ private — allowed via friend
}

int main() {
    BankAccount acc("Rahul", 50000);
    auditAccount(acc);         // ✅ works — it's a friend of BankAccount

    // acc.balance              ← ❌ still private to everyone else — friend doesn't affect this
}
```

---

## `friend` for `operator<<` — The Most Common Use

The `<<` operator needs `ostream` on the left side, so it can't be a member of your class. But it needs private data to print. Solution: `friend`:

```cpp
class Point {
    double x, y;   // private members
public:
    Point(double x, double y) : x(x), y(y) {}

    // operator<< must be non-member (ostream is the left operand)
    // needs x and y (private) → make it friend
    friend ostream& operator<<(ostream& os, const Point& p) {
        os << "(" << p.x << ", " << p.y << ")";   // accesses private x and y
        return os;   // return ostream to allow chaining: cout << p1 << p2
    }
};

int main() {
    Point p(3.0, 4.0);
    cout << p << "\n";         // (3, 4) — calls friend operator<<
    cout << p << " end\n";     // (3, 4) end — chaining works because we return os
}
```

---

## `friend` Class — Grant Access to an Entire Class

When one class needs to work deeply with another's internals:

```cpp
class Engine {
private:
    int  horsepower;
    bool running;

    friend class Car;   // ← Car class gets FULL access to Engine's private members

public:
    Engine(int hp) : horsepower(hp), running(false) {}
};

class Car {
    Engine engine;
public:
    Car(int hp) : engine(hp) {}

    void start() {
        engine.running    = true;       // ✅ private Engine member — allowed via friend
        cout << "Started: " << engine.horsepower << " HP\n";  // ✅ private — allowed
    }
};

int main() {
    Car c(200);
    c.start();    // Started: 200 HP

    // c.engine.running  ← ❌ ERROR: Car is friend of Engine, but you (in main) are not
    // friend access stays within Car — it doesn't propagate outward
}
```

---

## Key Properties of `friend` — The Rules

| Property | What It Means |
|---|---|
| **One-directional** | If A declares B a friend, B can access A's privates. A cannot access B's privates (unless B also declares A a friend). |
| **Not inherited** | If B is a friend of A, B's child class is NOT automatically a friend of A. Friendship does NOT pass to children. |
| **Not transitive** | If A is a friend of B, and B is a friend of C — A is NOT a friend of C. Friendship doesn't chain. |
| **Explicit grant** | Only what's declared a friend gets access. Nobody else is affected. |

---

## When to Use `friend` — Guidelines

```
✅ Appropriate uses:
   • operator<< and operator>> overloads (must be non-member, need private access)
   • Tightly coupled helper classes (Engine/Car, Node/LinkedList)
   • Unit test fixture classes that inspect private state for testing

❌ Avoid friend for:
   • Convenience — adding a proper getter/setter is cleaner design
   • Bypassing encapsulation just to avoid thinking about the API
   • Any case where a public interface would work just as well
```

---

## `friend` vs Getter — When to Choose What

```cpp
class BankAccount {
private:
    double balance;
public:
    // Option A: getter (clean public interface — preferred in most cases)
    double getBalance() const { return balance; }

    // Option B: friend (gives a specific external function read/write access to privates)
    // The friend can be DECLARED here and DEFINED outside the class (most common pattern):
    friend double auditBalance(const BankAccount& a);
};

// Definition outside the class
double auditBalance(const BankAccount& a) { return a.balance; }

// Alternative — "inline friend" (hidden friend): define the body directly inside the class declaration.
// The function still isn't a member, but it becomes findable ONLY via ADL (Argument-Dependent Lookup),
// not by direct unqualified call. This is used extensively in the STL and modern C++ libraries
// to prevent unintended matches in overload resolution.
//
// class BankAccount {
//     friend double auditBalance(const BankAccount& a) {  // defined inline — hidden friend
//         return a.balance;
//     }
// };
// auditBalance(acc);   // ✅ found via ADL (argument type is BankAccount)
// auditBalance(someInt); // ❌ NOT found — hidden from direct lookup
//
// For SDE-1 interviews: know that inline friends exist and are used in the STL;
// you don't need to use them yourself, but "what is a hidden friend?" is a real question.
```

**Interview Answer:** "`friend` deliberately breaks encapsulation to give a specific external function or class access to private members. It's non-inheritable, non-transitive, and one-directional. Use it for operator overloading and tightly-coupled helpers — not as a lazy substitute for a proper public interface."

---

# Part 2 — Namespaces: Preventing Name Collisions

## The Problem Without Namespaces

In a large project, you might use two libraries that both define a function called `add` or a variable called `PI`. Without namespaces, the compiler sees two conflicting definitions and fails:

```cpp
// math_lib.h
int add(int a, int b) { return a + b; }
double PI = 3.14159;

// physics_lib.h
int add(int a, int b) { return a + b; }   // SAME NAME — collision!
double PI = 3.14159;                       // SAME NAME — collision!

// main.cpp
#include "math_lib.h"
#include "physics_lib.h"
// LINKER ERROR: 'add' defined twice, 'PI' defined twice
```

**Namespaces** solve this by wrapping each library's symbols in its own named scope:

---

## Defining and Using Namespaces

```cpp
namespace Math {
    int add(int a, int b) { return a + b; }
    const double PI = 3.14159;

    class Vector {
    public:
        double x, y;
        Vector(double x, double y) : x(x), y(y) {}
    };
}

namespace Physics {
    int add(int a, int b) { return a + b; }   // same name — different scope, no conflict!
    const double PI = 3.14159;
}

int main() {
    int r1 = Math::add(2, 3);       // 5 — Math's add
    int r2 = Physics::add(2, 3);    // 5 — Physics's add (same result, different function)

    Math::Vector v(1.0, 2.0);       // Math's Vector class
    cout << Math::PI << "\n";        // 3.14159 — Math's PI
    cout << Physics::PI << "\n";     // 3.14159 — Physics's PI (different variable)
}
```

The `::` operator is the **scope resolution operator** — it says "look inside this namespace."

---

## `using` Directive and `using` Declaration

```cpp
// using declaration — bring ONE specific symbol into current scope
using Math::add;
add(2, 3);     // no prefix needed — refers to Math::add
Math::PI;      // PI still needs full qualification

// using directive — bring EVERYTHING from namespace into current scope
using namespace Math;
add(2, 3);     // Math::add (no prefix)
PI;            // Math::PI (no prefix)
Vector v;      // Math::Vector (no prefix)

// CAUTION: "using namespace std;" in global scope risks collision
// If your code and std:: both define something called 'count', it's ambiguous
// Preferred: either use full std:: prefix, or limit 'using namespace std;' to function scope
```

**In header files:** NEVER write `using namespace std;` (or any namespace) in a header. It forces that namespace into every file that includes the header — polluting their scope with names they didn't ask for.

---

## Nested Namespaces

```cpp
namespace Company {
    namespace Backend {
        void handleRequest() { cout << "Backend\n"; }
    }
    namespace Frontend {
        void handleRequest() { cout << "Frontend\n"; }
    }
}

Company::Backend::handleRequest();    // Backend
Company::Frontend::handleRequest();   // Frontend

// C++17 shorthand for defining nested namespaces:
namespace Company::Backend {
    void anotherFunction() { }
}
```

---

## Anonymous (Unnamed) Namespaces — File-Local Scope

An anonymous namespace makes its contents visible **only within the current file** — they have internal linkage, like C's `static` at file scope:

```cpp
namespace {
    void helperFunc() { cout << "Only visible in this .cpp file\n"; }
    int internalData = 42;
}

// In the same .cpp file:
helperFunc();   // ✅ works

// In other .cpp files:
// helperFunc() is invisible — not exported, no linker symbol
```

**Use case:** File-local helper functions that shouldn't pollute the global namespace or be callable from other files. Prefer this over C's `static` for file-local linkage.

---

## `namespace` vs `class` — Not the Same Thing

Both create a named scope, but they serve different purposes:

| Feature | `namespace` | `class` |
|---|---|---|
| Can instantiate? | No (`Math obj;` makes no sense) | Yes (`MyClass obj;`) |
| Access specifiers? | No — everything inside is accessible | Yes (`public`, `private`, `protected`) |
| Can be reopened? | Yes — can add members in multiple files | No — closed after definition |
| Inheritance? | No | Yes |
| Purpose | Group symbols, prevent collisions | Blueprint for objects + encapsulation |

---

## `std` Namespace

The entire C++ standard library lives inside the `std` namespace:

```cpp
std::cout << "Hello\n";          // std::cout — the standard output stream
std::vector<int> v;              // std::vector — dynamic array
std::string s = "hello";         // std::string — string type
std::sort(v.begin(), v.end());   // std::sort — sorting algorithm

// using namespace std; saves typing but risks collision with your own names
// Especially: NEVER write "using namespace std;" in a header file
```

---

## Namespace Best Practices

```
✅ DO:
   • Wrap each logical module in its own namespace (Math, Network, UI)
   • Use anonymous namespaces for file-local helpers instead of C's 'static'
   • In headers: always use fully qualified names (std::vector, not just vector)
   • Keep namespace nesting shallow (2 levels max)

❌ DON'T:
   • Never write "using namespace std;" in a header — pollutes every includer's scope
   • Don't use deeply nested namespaces (Company::Backend::Auth::Service::...) — verbose
   • Don't use namespaces as a substitute for proper class design
```

---

## Interview Q&A

**Q: "What is a `friend` function?"**
A: A non-member function declared inside a class with `friend`, granting it access to that class's private and protected members. Used primarily for operator overloading and tightly-coupled utilities.

**Q: "Is friendship inherited?"**
A: No. Friendship is not inherited, not transitive, and not bidirectional. Only the explicitly declared entity gets access.

**Q: "Why use `friend` for `operator<<`?"**
A: `operator<<` needs `ostream` as the left operand, so it must be a non-member function. But it needs private members to print them. `friend` gives it the access it needs without exposing those members publicly.

**Q: "What is a namespace?"**
A: A named scope that groups related identifiers (functions, classes, variables) to prevent naming collisions. Members are accessed with `NamespaceName::identifier` or imported with `using`.

**Q: "What is an anonymous namespace?"**
A: A namespace with no name. Its contents are visible only within the current translation unit (file). Modern replacement for C's file-scope `static` linkage.

**Q: "Why not write `using namespace std;` in a header?"**
A: It forces the entire `std` namespace into the scope of every file that includes that header, potentially causing naming conflicts that are hard to trace and that the includer can't control or opt out of.

**Q: "What does `::` mean?"**
A: The scope resolution operator. It accesses a member of a specific namespace or class. `Math::PI` = the `PI` inside the `Math` namespace. `ClassName::member` = a member of that class (e.g., a static member).
