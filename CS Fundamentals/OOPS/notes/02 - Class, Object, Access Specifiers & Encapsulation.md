# Part 1 — Class, Object & Access Specifiers

## The Full Class Syntax — Anatomy

Every class follows this skeleton. Read it once and the rest makes sense:

```cpp
class ClassName {
    private:
        // Members here are ONLY accessible INSIDE this class
        // (not from main(), not from child classes)

    protected:
        // Members here are accessible INSIDE this class
        // AND inside any child (derived) class
        // but NOT from outside (not from main())

    public:
        // Members here are accessible from ANYWHERE
        // (inside class, child class, main(), other classes)

};   // ← This semicolon is mandatory — forgetting it causes cryptic errors
```

> **Default in C++:** If you don't write any access specifier, everything defaults to `private`. This is the opposite of `struct`, where the default is `public`.

---

## Access Specifiers — The Core Rules

Think of it as a "who can see this?" question:

| Specifier | Inside Same Class | Inside Child Class | Outside (main/others) |
|---|---|---|---|
| `private` | ✅ Yes | ❌ No | ❌ No |
| `protected` | ✅ Yes | ✅ Yes | ❌ No |
| `public` | ✅ Yes | ✅ Yes | ✅ Yes |

**The key insight:** `private` is for things only *this* class should touch. `protected` is for sharing with *children* only. `public` is for the *world*.

---

## Building a Real Class — Step by Step

Let's build a `Teacher` class that properly hides sensitive data (salary) but exposes safe operations:

```cpp
#include <iostream>
#include <string>
using namespace std;

class Teacher {
    private:
        double salary;   // HIDDEN from outside — sensitive data
                         // Only teacher's own methods can read/write this

    public:
        string name;        // Accessible from anywhere — not sensitive
        string department;
        string subject;

        // SETTER — a controlled "write door" for private data
        // You can add validation here (e.g., salary can't be negative)
        void setSalary(double s) {
            if (s >= 0)          // validation — direct field access can't do this
                salary = s;
        }

        // GETTER — a controlled "read door" for private data
        // Caller can read salary, but can't directly change it
        double getSalary() {
            return salary;
        }

        void changeDepartment(string newDept) {
            department = newDept;
        }
};

int main() {
    Teacher t1;                       // object created — memory allocated

    t1.name       = "Shraddha";       // ✅ public — direct access fine
    t1.department = "Computer Science";
    t1.subject    = "C++";
    t1.setSalary(25000);              // ✅ controlled write via setter

    cout << t1.name << endl;          // ✅ public — direct read fine
    cout << t1.getSalary() << endl;   // ✅ controlled read via getter

    // t1.salary = 99999;  ← ❌ ERROR: salary is private — compiler blocks this
    return 0;
}
```

---

## Why Not Just Make Everything `public`?

You might wonder: why bother hiding salary? Just make it public.

Here's the problem:

```cpp
// If salary is public:
t1.salary = -50000;   // completely valid code — no way to stop this nonsense

// If salary is private with a setter:
t1.setSalary(-50000);  // setter catches it: if (s >= 0) → rejected!
```

Setters let you **validate and control** what data gets written. Direct field access bypasses all guards. This is the entire point of access specifiers.

---

## Getters & Setters — The Pattern

This pattern is universally used. Here's a more complete example with a BankAccount:

```cpp
class BankAccount {
    private:
        double balance;    // hidden — can't be set arbitrarily
        string password;   // hidden — never exposed AT ALL (no getter!)

    public:
        // Setter with validation
        void setBalance(double amount) {
            if (amount >= 0)       // can't have negative balance
                balance = amount;
            // if amount < 0: silently ignored here for brevity
            // In real code, always throw an exception (e.g. std::invalid_argument) instead
            // Silent failure is dangerous — the caller has no way to know the write was rejected
        }

        // Getter — read-only view of private data
        double getBalance() const {   // 'const' = this function won't modify anything
            return balance;
        }

        // password has NO getter and NO setter — it's completely inaccessible
        // from outside the class. This is intentional security.
};
```

**Rule of thumb:** Ask "does the outside world *need* to read/write this?" If no — make it private. If yes — add only the getter/setter needed, not both blindly.

---

# Part 2 — Encapsulation

## Encapsulation — What It Actually Means

> **Encapsulation** = bundling related **data members** and **member functions** that operate on that data into a **single unit** (the class).

The analogy: a medicine capsule bundles multiple ingredients into one unit. You interact with the capsule, not the raw ingredients.

```
Data Members  ──┐
                ├──→ CLASS (the capsule) = Encapsulation
Member Funcs  ──┘
```

### Encapsulation ≠ Just Creating a Class

Creating any class is encapsulation. But *good* encapsulation means:
1. Related data and functions are grouped together logically
2. Sensitive data is hidden using `private`/`protected`
3. Only a minimal, well-defined public interface is exposed

### Encapsulation Enables Data Hiding

```cpp
class Account {
    private:
        double balance;   // DATA HIDING — cannot be touched from outside
        string password;  // DATA HIDING — sensitive, locked away

    public:
        int accountId;    // NOT hidden — needed publicly
        string username;  // NOT hidden — needed publicly
};
```

**Data hiding** = the practice of marking sensitive members as `private` or `protected`, preventing unauthorized access. It's the *result* of doing encapsulation properly.

---

## Encapsulation vs Data Hiding — They're Related But Different

| Concept | What It Does | Mechanism |
|---|---|---|
| **Encapsulation** | Groups data + functions into a class | The `class` keyword itself |
| **Data Hiding** | Restricts access to sensitive members | `private` / `protected` keywords |

> Data hiding is a *consequence* of good encapsulation. You can have encapsulation without data hiding (everything public), but you can't have data hiding without encapsulation.

---

## `struct` vs `class` — The Only Real Difference

```cpp
struct S {
    int x;    // PUBLIC by default — same as writing 'public: int x;'
    int y;
};

class C {
    int x;    // PRIVATE by default — same as writing 'private: int x;'
    int y;
};

// They're functionally identical — the default access level is all that differs
S s; s.x = 5;    // ✅ works — struct members are public
C c; c.x = 5;    // ❌ ERROR — class members are private by default
```

**Convention:**
- Use `struct` for plain data containers (just data, minimal or no behaviour)
- Use `class` for full OOPs entities with encapsulation, validation, methods

---

## Multiple Classes in One Program

Classes can coexist. Each object gets its own independent memory block:

```cpp
class Student {
    public:
        string name;
        int rollNo;
        int age;
};

class Teacher {
    private:
        double salary;
    public:
        string name;
        string dept;
        void setSalary(double s) { salary = s; }
        double getSalary() { return salary; }
};

int main() {
    Student s1;           // s1 gets memory: name + rollNo + age
    Student s2;           // s2 gets its OWN memory: completely separate from s1
    Teacher t1;           // t1 gets memory: name + dept + salary

    s1.name = "Rahul";
    s2.name = "Neha";     // s2.name is independent — changing it doesn't touch s1.name

    t1.setSalary(30000);
    // t1.salary = 30000; ← ERROR: private
}
```

---

## Memory Layout — What Happens in RAM

```
Stack memory (each object = its own block):

┌──────────────────┐  ← Teacher t1 (starts here)
│  name  (string)  │
│  dept  (string)  │
│  salary(double)  │
└──────────────────┘

┌──────────────────┐  ← Teacher t2 (entirely separate block)
│  name  (string)  │
│  dept  (string)  │
│  salary(double)  │
└──────────────────┘

t1.name = "A" has ZERO effect on t2.name — they live at different addresses
```

---

## Dot (`.`) vs Arrow (`->`) Operator

```cpp
Teacher t1;           // stack object
t1.name = "Shraddha"; // DOT operator — use when object is not a pointer

Teacher* ptr = new Teacher();  // heap object — ptr is a pointer to the object
ptr->name = "Shraddha";        // ARROW operator — use when you have a pointer
                                // ptr->name is shorthand for (*ptr).name
delete ptr;           // must manually free heap objects
```

**Rule:** If it's a regular variable → use `.`  |  If it's a pointer → use `->`

---

## Interview Q&A — Memorise These

**Q: "What is encapsulation?"**
A: Wrapping of data members and member functions into a single unit called a class. See [[01 - Introduction, OOP & Pointers#The Two Fundamental Concepts|Class vs Object]] for foundational concepts.

**Q: "What is data hiding?"**
A: Restricting access to sensitive data members by marking them `private` or `protected`, preventing unauthorized modification from outside the class.

**Q: "What are access specifiers?"**
A: Keywords (`private`, `protected`, `public`) that control which parts of the program can access a class's members.

**Q: "What is a getter/setter?"**
A: Public member functions that provide controlled read (`get`) and controlled write (`set`) access to private data members. Setters can include validation logic.

**Q: "Difference between `private` and `protected`?"**
A: `private` blocks ALL external access, including child classes. `protected` allows access from child (derived) classes but blocks everything else outside.

**Q: "Difference between encapsulation and data hiding?"**
A: Encapsulation is the concept of bundling data + functions into a class. Data hiding is specifically about restricting access to sensitive members using `private`/`protected`. Data hiding is a result of good encapsulation.

**Q: "What's the default access in class vs struct?"**
A: In `class`, default is `private`. In `struct`, default is `public`.
