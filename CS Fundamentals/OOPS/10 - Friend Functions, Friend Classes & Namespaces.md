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

## C++ Operator Overloading: `operator<<` and the `friend` Keyword

### Why `operator<<` Must Be a Non-Member Function
When you use `cout << p;`, the compiler evaluates this expression as:
`operator<<(cout, p)`
* **Left Operand:** `std::ostream` (`cout`)
* **Right Operand:** `Point` (`p`)
<u>If you want to overload an operator as a member function, the left operand must be an object of that class</u>. Since you cannot modify the standard `std::ostream` class to add a member function for your custom `Point` class, `operator<<` **must be implemented as a non-member function**.
Because `operator<<` is a non-member function, it cannot access the private data members (`x`, `y`) of the `Point` class by default.
### Solution 1: Using a `friend` Function
Declaring the function as a `friend` inside the class grants it direct access to the class's private members. 
> **Note:** Even when defined inside the class body, a `friend` function remains a non-member function.

```cpp
class Point {
private:
    double x, y;
public:
    Point(double x, double y) : x(x), y(y) {}
    // Grant access to private members and define the function
    friend std::ostream& operator<<(std::ostream& os, const Point& p) {
        os << "(" << p.x << ", " << p.y << ")";
        return os;   // Returns os to enable method chaining (e.g., cout << p1 << p2;)
    }
};
```
#### The Problem Without `friend`
If you remove the `friend` keyword and try to implement it as a standard non-member function, compilation will fail:
C++
```cpp
// Compilation Error: x and y are private
std::ostream& operator<<(std::ostream& os, const Point& p) {
    os << p.x;   // ERROR: x is private within this context
    return os;
}
```
### Solution 2: Alternative Without `friend` (Using Public Getters)
If you prefer to keep encapsulation strict without using `friend`, you must expose the private data through public getter methods.
C++
```cpp
class Point {
private:
    double x, y;
public:
    Point(double x, double y) : x(x), y(y) {}
    // Public getters
    double getX() const { return x; }
    double getY() const { return y; }
};
// Non-member function utilizing public interface
std::ostream& operator<<(std::ostream& os, const Point& p) {
    os << "(" << p.getX() << ", " << p.getY() << ")";
    return os;
}
```

---
## `friend` Function & Class

### Friend Function
Access to private members to a non-member fucntion
```cpp
class Engine {
private:
	int horsepower;
	// friend function declaration
	friend void upgradeEngine(Engine& e);
public:
	Engine(int hp) : horsepower(hp) {}
	void display() const { std::cout << horsepower << " HP\n"; }
};
// Definition of the friend function (not a member of the class)
void upgradeEngine(Engine& e) {
	e.horsepower += 50; // ✅ Direct access to private member
}
```

Access to private members to another Class
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
        engine.running    = true; // ✅ private Engine member
        cout << "Started: " << engine.horsepower << " HP\n";  // ✅ 
    }
};
```

- Declaring a `friend class X;` or `friend void func();` automatically acts as a forward declaration, meaning no prior declaration of the class or function is required.
- `frined` can be declared in both private and public scope (no difference).

- When making a specific member function of **Class A** a friend of **Class B**, a forward declaration of **Class B** is required to break the circular dependency.
- You cannot reference a member of a class (`Mechanics::tuneUp`) unless that class and its member have already been fully declared.
- A `friend` function defined inside a class body is a true non-member (global) function that executes without a dot-notation syntax.
```cpp
class Car; // 1. Forward declaration required for function parameter below
class Mechanics {
public:
	void tuneUp(Car&); // 2. can't contain definition as performance not defined
};
class Car {
private:
	int performance = 100;
	friend void Mechanics::tuneUp(Car& c); // 3. Grants friendship to specific function
};
void Mechanics::tuneUp(Car& c) {
	c.performance += 20; // 4. Defined last so it can access Car's private members
}
```

---

## Key Properties of `friend` — The Rules

| Property            | What It Means                                                                                                          |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| **One-directional** | If A declares B a friend, B can access A's privates. A cannot access B's privates (unless B also declares A a friend). |
| **Not inherited**   | If B is a friend of A, B's child class is NOT automatically a friend of A. Friendship does NOT pass to children.       |
| **Not transitive**  | If A is a friend of B, and B is a friend of C — A is NOT a friend of C. Friendship doesn't chain.                      |


---

## `friend` vs Getter — When to Choose What

**Interview Answer:** "`friend` deliberately breaks encapsulation to give a specific external function or class access to private members. It's non-inheritable, non-transitive, and one-directional. Use it for operator overloading and tightly-coupled helpers — not as a lazy substitute for a proper public interface." Getters are preferred as it keeps changes localized inside the class, ensuring internal variable renames never break external code.

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

The `::` operator is the **scope resolution operator** — it says "look inside this namespace. or class"

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
You *cannot explicitly "remove,"* "undeclare," or cancel a `using namespace`

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
Company::Frontend::handleRequest();   // Frontend (top-down)
// C++17 shorthand for defining nested namespaces:
namespace Company::Backend {
    void anotherFunction() { }
}
```

---

## Anonymous (Unnamed) Namespaces — File-Local Scope

An anonymous namespace makes its contents visible **only within the current file** — they have internal linkage.

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
**Use case:** File-local helper functions that shouldn't pollute the global namespace or be callable from other files.

---

## `namespace` vs `class` — Not the Same Thing
Both create a named scope, but they serve different purposes:

| Feature            | `namespace`                               | `class`                                |
| ------------------ | ----------------------------------------- | -------------------------------------- |
| Can instantiate?   | No (`Math obj;` makes no sense)           | Yes (`MyClass obj;`)                   |
| Access specifiers? | No — everything inside is accessible      | Yes (`public`, `private`, `protected`) |
| Can be reopened?   | Yes — can add members in multiple section | No — closed after definition           |
| Inheritance?       | No                                        | Yes                                    |
| Purpose            | Group symbols, prevent collisions         | Blueprint for objects + encapsulation  |

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
