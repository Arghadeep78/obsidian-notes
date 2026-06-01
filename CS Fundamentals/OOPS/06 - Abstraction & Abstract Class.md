# Part 1 — Abstraction & Abstract Class Basics

## What Is Abstraction? (The ATM Analogy)

When you use an ATM, you press "Withdraw ₹500". You don't know — and don't need to know — how the ATM connects to the bank server, verifies your balance, communicates with the cash dispenser, and logs the transaction. All of that is **hidden**. You only see and interact with the essential interface.

That's abstraction.

> **Abstraction** = hiding the **all unnecessary details(how)** (internal implementation details) and exposing only the **important parts(what)** (essential interface the user needs).

**Two sides of abstraction:**
1. **Hide** — unnecessary/sensetive complexity goes in `private`/`protected`
2. **Show** — only what the caller needs goes in `public`

---

## Abstraction vs Encapsulation vs Data Hiding — Clarified
These three are related but distinct:

| Concept                                                                                                             | What It Does                                                    | The Mechanism                        |
| ------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------- | ------------------------------------ |
| **[[02 - Class, Object, Access Specifiers & Encapsulation#Encapsulation — What It Actually Means\|Encapsulation]]** | Bundles data + functions into one unit (class)                  | The `class` keyword                  |
| **Data Hiding**                                                                                                     | Restricts access to sensitive internal data                     | `private` / `protected`              |
| **Abstraction**                                                                                                     | Hides implementation complexity, shows only essential interface | Access specifiers + abstract classes |

> **Easy way to remember:** Encapsulation is *bundling*. Data hiding is *locking*. Abstraction is *simplifying the view*.
> Encapsulation is the technique. Data hiding is the security outcome. Abstraction is the design goal.

---

## Way 1 — Abstraction via Access Specifiers

The simplest form: hide the *how*, expose the *what*:

```cpp
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


An **abstract class** is a class that:
 - *Cannot be instantiated* (you cannot create objects of it directly)
 - Exists purely to provide a *template/contract* for derived classes
 - Contains **at** **least** **one** *pure virtual function

	- Can have concrete methods → Pure virtual functions are not mandatory for every method.
	- Can have data members → Normal member variables are allowed.
	- Can have constructors → Called by derived class constructors.
	- Derived class must implement all pure virtual functions → Otherwise it remains abstract.
	- Pointers/references are allowed → `Shape* ptr = new Circle();` ✔️

A **pure virtual function** is a virtual function declared with `= 0` in the base class and has no implementation requirement at that level. It *forces* derived classes to provide their *own* *implementation*.
	See [[05 - Polymorphism — Overloading, Overriding & Virtual Functions#Function Overriding & Virtual Functions|Virtual Functions]] for details.

```cpp
class Animal { // abstract class
public:
    virtual void speak() = 0;   // Pure virtual function
};
class Dog : public Animal {
public:
    void speak() override {
        cout << "Woof!\n";
    }
};
int main() {
    // Animal a;      // ❌ Error: Cannot instantiate abstract class
    Animal* ptr = new Dog();   // ✅ Allowed
    ptr->speak();              // Output: Woof!
    delete ptr;
}
```

#### The key benefit:
abstract classes catch missing implementations **at compile time**, not at runtime.

```cpp
class Animal {
    public:
        virtual void speak() = 0;   // every animal MUST implement speak()
};
class Dog  : public Animal { void speak() override { cout << "Woof\n"; } };
class Fish : public Animal { };   // ❌ COMPILE ERROR — Fish doesn't implement speak()!
// Error: "cannot declare variable of abstract type 'Fish'"

// Without abstract class:
// class Fish { } would compile fine, and the missing speak() would be a runtime surprise
```

# Part 2 — Interfaces, Design Patterns & Edge Cases

## Interface vs Abstract Class (C++ Perspective)

C++ has no `interface` keyword (Java does). In C++, an **interface** is simulated by making an abstract class where **every method is pure virtual** and there are no data members.

```cpp
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

You can make a destructor pure virtual (to make a class abstract when it has no other pure virtual functions). But unlike other pure virtual functions, it **MUST have a body**(actual implementation or definition of the function):

```cpp
class Resource {
    public:
        virtual ~Resource() = 0;   // pure virtual — makes class abstract
};
Resource::~Resource() { //empty or code}   // BODY IS MANDATORY — explained below
                            // Even if it is empty {}, it must exist!

class File : public Resource {
    public:
        ~File() { cout << "File closed\n"; }
};
// Resource r;  ← ERROR: abstract
// File f;      ← OK: ~File runs, then Resource::~Resource body runs automatically
```

**Why the body is mandatory:** Destructors always run in chain — when `File` is destroyed, the compiler automatically calls `Resource::~Resource()` after `~File()` finishes. If that base destructor body didn't exist, the chain would have nothing to call → linker error.

#### Why the Body is Mandatory vs. Optional
- **Pure Virtual Destructor (Mandatory Body):** Destructors always execute in a bottom-up chain. When a derived object is destroyed, `~Derived()` runs first, and the compiler automatically calls `~Base()` immediately after. Without a base body, this destruction chain breaks, causing a **linker error**.
- **Regular Pure Virtual Functions (Optional Body):** The body is optional. It is only required if you explicitly bypass virtual dispatch and invoke it using the scope resolution operator (`Base::pureVirtualFn()`). Calling it explicitly without a body results in a **linker error**.

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
