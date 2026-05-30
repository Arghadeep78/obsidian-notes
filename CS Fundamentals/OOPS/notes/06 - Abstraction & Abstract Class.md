# Part 1 — Abstraction & Abstract Class Basics

## What Is Abstraction? (The ATM Analogy)

When you use an ATM, you press "Withdraw ₹500". You don't know — and don't need to know — how the ATM connects to the bank server, verifies your balance, communicates with the cash dispenser, and logs the transaction. All of that is **hidden**. You only see and interact with the essential interface.

That's abstraction.

> **Abstraction** = hiding the **how** (internal implementation details) and exposing only the **what** (essential interface the user needs).

**Two sides of abstraction:**
1. **Hide** — unnecessary complexity goes in `private`/`protected`
2. **Show** — only what the caller needs goes in `public`

---

## Abstraction vs Encapsulation vs Data Hiding — Clarified

These three are related but distinct:

| Concept | What It Does | The Mechanism |
|---|---|---|
| **[[02 - Class, Object, Access Specifiers & Encapsulation#Encapsulation — What It Actually Means\|Encapsulation]]** | Bundles data + functions into one unit (class) | The `class` keyword |
| **Data Hiding** | Restricts access to sensitive internal data | `private` / `protected` |
| **Abstraction** | Hides implementation complexity, shows only essential interface | Access specifiers + abstract classes |

> **Easy way to remember:** Encapsulation is *bundling*. Data hiding is *locking*. Abstraction is *simplifying the view*.
> Encapsulation is the technique. Data hiding is the security outcome. Abstraction is the design goal.

---

## Way 1 — Abstraction via Access Specifiers

The simplest form: hide the *how*, expose the *what*:

```cpp
#include <iostream>
#include <string>
using namespace std;

class BankAccount {
    private:
        double balance;        // HOW the balance is stored — hidden from caller
        string password;       // internal secret — never exposed

    public:
        int accountId;         // WHAT you need to identify the account
        string username;

        // Caller says "deposit ₹5000" — doesn't need to know how balance changes internally
        void deposit(double amount) {
            if (amount > 0) balance += amount;  // validation logic hidden inside
        }

        // Caller can check balance — but can't directly set it
        double getBalance() {
            return balance;
        }
};

int main() {
    BankAccount acc;
    acc.accountId = 1001;
    acc.username  = "Rahul";
    acc.deposit(5000);
    cout << acc.getBalance() << endl;   // 5000
    // acc.balance = 99999;  ← ❌ ERROR — the implementation is abstracted away
}
```

The caller uses the account without knowing *how* balance is stored or validated internally.

---

## Way 2 — Abstraction via Abstract Class

### The Need: Enforcing a Contract

Suppose you're building a graphics library. You have `Circle`, `Rectangle`, `Triangle` — all are shapes. Each *must* implement `draw()` and `area()`. Without any mechanism to enforce this, a developer could forget to implement `draw()` in a new shape class, and the bug would only appear at runtime.

**Abstract class** solves this: make `draw()` a **contract** — if you don't implement it, your class can't be instantiated. The compiler *forces* compliance.

> An **abstract class** is a class that:
> - **Cannot be instantiated** (you cannot create objects of it directly)
> - Exists purely to provide a **template/contract** for derived classes
> - Contains at least one **pure virtual function**

---

### Pure Virtual Function

A **pure virtual function** is declared with `= 0`. It has **no implementation** in the base class. Any derived class that inherits it MUST provide an implementation — or it also becomes abstract. See [[05 - Polymorphism — Overloading, Overriding & Virtual Functions#Virtual Functions — The Core of Runtime Polymorphism|virtual functions]] for details.

```cpp
virtual void draw() = 0;   // pure virtual — "= 0" means no body, must override
```

Any class with at least one pure virtual function is automatically abstract.

---

## Complete Example — Abstract Class in Action

```cpp
#include <iostream>
using namespace std;

// ABSTRACT CLASS — cannot instantiate Shape directly
// Acts as a contract: every shape MUST implement draw() and area()
class Shape {
    public:
        virtual void draw()        = 0;   // pure virtual — no body here
        virtual double area()      = 0;   // pure virtual — no body here

        void describe() {                  // CONCRETE method — has a body, shared by all shapes
            cout << "I am a shape" << endl;
        }
        // Abstract class CAN have: constructors, data members, concrete methods
};

class Circle : public Shape {
    private:
        double radius;
    public:
        Circle(double r) : radius(r) {}

        void draw() override {             // MUST implement — or Circle is also abstract
            cout << "Drawing Circle with radius " << radius << endl;
        }

        double area() override {           // MUST implement
            return 3.14159 * radius * radius;
        }
};

class Rectangle : public Shape {
    private:
        double width, height;
    public:
        Rectangle(double w, double h) : width(w), height(h) {}

        void draw() override {
            cout << "Drawing Rectangle " << width << "x" << height << endl;
        }

        double area() override {
            return width * height;
        }
};

int main() {
    // Shape s;       ← ❌ ERROR: cannot instantiate abstract class
    //                   compiler says: "Shape is abstract because draw() is not overridden"

    Circle    c(5);
    Rectangle r(4, 6);

    c.draw();              // Drawing Circle with radius 5
    r.draw();              // Drawing Rectangle 4x6
    cout << c.area();      // 78.5397
    cout << r.area();      // 24

    c.describe();          // I am a shape — inherited concrete method works fine

    // Polymorphism: base pointer → derived object (classic pattern)
    Shape* s1 = new Circle(3);
    Shape* s2 = new Rectangle(2, 8);
    s1->draw();            // Drawing Circle with radius 3
    s2->draw();            // Drawing Rectangle 2x8

    delete s1; delete s2;
}
```

---

## Properties of Abstract Classes

| Property | Detail |
|---|---|
| **Cannot instantiate** | `Shape s;` is a compile error |
| **Can have concrete methods** | Not everything needs to be pure virtual |
| **Can have data members** | Normal member variables are fine |
| **Can have constructors** | Called by derived class constructors via initializer list |
| **Derived class obligation** | Must implement ALL pure virtual functions, or it's also abstract |
| **Pointer/reference is fine** | `Shape* ptr = new Circle();` is perfectly valid |

---

## Why Abstract Classes? — Compile-Time Contract Enforcement

```cpp
class Animal {
    public:
        virtual void speak() = 0;   // every animal MUST implement speak()
};

class Dog  : public Animal { void speak() override { cout << "Woof\n"; } };
class Cat  : public Animal { void speak() override { cout << "Meow\n"; } };
class Fish : public Animal { };   // ❌ COMPILE ERROR — Fish doesn't implement speak()!
// Error: "cannot declare variable of abstract type 'Fish'"

// Without abstract class:
// class Fish { } would compile fine, and the missing speak() would be a runtime surprise
```

**The key benefit:** abstract classes catch missing implementations **at compile time**, not at runtime.

---

# Part 2 — Interfaces, Design Patterns & Edge Cases

## Abstract Class with Shared Logic — Real Design Pattern

Abstract classes can mix pure virtual (must override) and concrete (shared) methods:

```cpp
class Database {
    public:
        // CONTRACT — every database must implement these
        virtual void connect()         = 0;
        virtual void disconnect()      = 0;
        virtual void query(string sql) = 0;

        // SHARED LOGIC — common to all databases, no override needed
        void logQuery(string sql) {
            cout << "[LOG] Executing: " << sql << endl;
        }
};

class MySQL : public Database {
    public:
        void connect()    override { cout << "MySQL connected\n"; }
        void disconnect() override { cout << "MySQL disconnected\n"; }
        void query(string sql) override {
            logQuery(sql);    // reuse the shared logic from base
            cout << "MySQL: " << sql << endl;
        }
};

class PostgreSQL : public Database {
    public:
        void connect()    override { cout << "PG connected\n"; }
        void disconnect() override { cout << "PG disconnected\n"; }
        void query(string sql) override {
            logQuery(sql);    // same shared logger
            cout << "PG: " << sql << endl;
        }
};
```

---

## Interface vs Abstract Class (C++ Perspective)

C++ has no `interface` keyword (Java does). In C++, an **interface** is simulated by making an abstract class where **every method is pure virtual** and there are no data members.

```cpp
// ABSTRACT CLASS — partial contract + shared implementation
class Animal {
    protected:
        string name;           // has data members
    public:
        Animal(string n) : name(n) {}
        virtual void speak() = 0;    // pure virtual — must override
        void breathe() {             // CONCRETE — shared by all animals
            cout << name << " is breathing\n";
        }
};

// INTERFACE (simulated in C++) — pure contract only
class Flyable {
    public:
        virtual void fly()  = 0;    // all pure virtual
        virtual void land() = 0;    // all pure virtual
        virtual ~Flyable()  = 0;    // virtual destructor — needed if deleting via Flyable*
        // NO data members, NO concrete methods
};
Flyable::~Flyable() {}   // body required even for pure virtual destructor
```

| Feature | Abstract Class | Interface (C++ simulation) |
|---|---|---|
| Pure virtual functions | At least one | ALL methods |
| Data members | Allowed | None |
| Concrete methods | Allowed | None |
| Purpose | "is-a" with shared code | Capability contract |
| Real-world analogy | `Animal` (shared traits) | `Flyable`, `Serializable` (abilities) |

**Use abstract class** when you have common behaviour to share (e.g., `Database::logQuery`).
**Use interface** when you just want to say "any class that implements this has this capability" across unrelated types (e.g., `Car` and `BankAccount` can both be `Serializable`).

---

## Partial Implementation — Derived Class Also Abstract

If a derived class doesn't implement ALL pure virtual functions, it also becomes abstract:

```cpp
class Shape      { virtual void draw() = 0; virtual double area() = 0; };
class PartialShape : public Shape {
    void draw() override { cout << "Drawing\n"; }
    // area() still not overridden → PartialShape is ALSO abstract
};
class ConcreteCircle : public PartialShape {
    double area() override { return 3.14; }   // now all pure virtuals are overridden → concrete
};
```

---

## Pure Virtual Destructor — The Special Case

You can make a destructor pure virtual (to make a class abstract when it has no other pure virtual functions). But unlike other pure virtual functions, it **MUST have a body**:

```cpp
class Resource {
    public:
        virtual ~Resource() = 0;   // pure virtual — makes class abstract
};
Resource::~Resource() { }   // BODY IS MANDATORY — explained below

class File : public Resource {
    public:
        ~File() { cout << "File closed\n"; }
};

// Resource r;  ← ERROR: abstract
// File f;      ← OK: ~File runs, then Resource::~Resource body runs automatically
```

**Why the body is mandatory:** Destructors always run in a chain — when `File` is destroyed, the compiler automatically calls `Resource::~Resource()` after `~File()` finishes. If that base destructor body didn't exist, the chain would have nothing to call → linker error.

**Two distinct rules — don't confuse them:**
- For **any pure virtual function other than the destructor:** the body is *optional*. If you provide one, it can be called explicitly via `Base::pureVirtualFn()`. If you don't provide one and never call it explicitly, no problem — but explicitly calling a pure virtual with no body is a linker error.
- For the **pure virtual destructor specifically:** the body is *mandatory*. Destructors always chain — when a derived object is destroyed, the compiler automatically calls every base destructor in order. If `Resource::~Resource()` had no body at all, the linker would error because there is nothing to call at the end of that chain.

The `= 0` still makes the class abstract (preventing `Resource r;`) — the body does not change that.

---

## Interview Q&A

**Q: "What is abstraction?"**
A: Hiding unnecessary implementation details while exposing only the essential interface. Example: a car's driver uses the steering wheel and pedals without knowing the internal mechanics.

**Q: "What is an abstract class?"**
A: A class with at least one pure virtual function that cannot be instantiated. It defines a contract that all derived classes must fulfil.

**Q: "What is a pure virtual function?"**
A: A virtual function assigned `= 0`, with no implementation in the base class. Every concrete derived class must override it, otherwise it's also abstract.

**Q: "Can an abstract class have a constructor?"**
A: Yes. It's called by derived class constructors via the initializer list to initialise the base portion.

**Q: "Can you create a pointer to an abstract class?"**
A: Yes. `Shape* ptr = new Circle();` is valid. You can't create a `Shape` object directly, but a pointer to it works fine and enables runtime polymorphism.

**Q: "What happens if a derived class doesn't override all pure virtual functions?"**
A: The derived class also becomes abstract and cannot be instantiated.

**Q: "Difference between abstraction and encapsulation?"**
A: Encapsulation bundles data and functions into a class. Abstraction hides the implementation complexity and shows only the necessary interface. Encapsulation is the mechanism; abstraction is the design goal it enables.
