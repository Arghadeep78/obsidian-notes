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
        // AND inside any derived (child) class
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

**The key insight:** `private` is for things only *self* class should touch. `protected` is for sharing with *children* only. `public` is for the *world*.

---

## Building a Real Class — Step by Step

Let's build a `Teacher` class that properly hides sensitive data (salary) but exposes safe operations:

```cpp
#include <iostream>
#include <string>
using namespace std;

class Teacher {
private:
    double salary;   // Hidden from outside intentional security
public:
    string name;
    string subject;
    string password // no getter or setter
    void setSalary(double amount) {  // Setter → controlled write access
	    if(amount>=0)
	        salary = amount;
	    else
		    cout<<"Enter Valid Salary\n";
    }
    double getSalary() {        // Getter → controlled read access
        return salary;
    }
    double getName() const {
	    cout<<name<<endl;   // 'const' = this function won't modify anything
};
int main() {
    Teacher t1;
    t1.name = "Arghadeep";
    t1.subject = "C++";
    t1.setSalary(25000);   // Set private data
    cout << t1.name << endl;
    cout << t1.subject << endl;
    cout << t1.getSalary() << endl;  // Get private data

    // t1.salary = 50000;  // ❌ Error (private)

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

eg: setter may take in a passcode

Therefore we use *Getters* and *Setter*.

**Rule of thumb:** Ask "does the outside world *need* to read/write this?" If no — make it private. If yes — add only the getter/setter needed, not both blindly.

---

# Part 2 — Encapsulation

> ***Encapsulation*** = wrapping up of related **data members** and **member functions** that operate on that data into a **single unit** (the class).

The analogy: a medicine capsule bundles multiple ingredients into one unit. You interact with the capsule, not the raw ingredients.

```
Data Members  ──┐
                ├──→ CLASS (the capsule) = Encapsulation
Member Funcs  ──┘
```

### Encapsulation ≠ Just Creating a Class

Creating any class is encapsulation. But *good* encapsulation means:
1. Related data and functions are <u>grouped</u> together logically (*classes* and *sub-class*)
2. Sensitive data is hidden using `private`/`protected`
3. Only a minimal, well-defined public interface is exposed

### Encapsulation Enables *Data Hiding*

```cpp
class Account {
    private:
        double balance;   // DATA HIDING — cannot be touched from outside
        string password;  // DATA HIDING — sensitive, locked away

    public:
        int accountId; 
           // NOT hidden — needed publicly
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

> Data hiding is a *<u>consequence</u>* of good encapsulation. You can have encapsulation without data hiding (everything public), but you can't have data hiding without encapsulation.

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
- Use `struct` for simple data containers (just data, minimal or no behaviour, non-confidential data)
- Use `class` for full OOPs entities with encapsulation, validation, methods

- Struct also have *protected* access modifier.
- Struct has mostly stayed due to *historical* reasons. *C* had struct without OOPs.

---

## Multiple Classes in One Program

Classes can coexist. Each object gets its own independent memory block:

An *object* is an **instance of a class**.

A class is a *blueprint*, while an object is the **actual thing created from that blueprint** and stored in memory.

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

<u>Each object</u> gets its **own copy** of all *non-static* data members. (check: [[07 - The static Keyword in C++ OOPs#1. Static Local Variable — Remembers Between Calls | Static Keyword]])
Changing `t1` does not affect `t2` because they occupy different memory locations.
Different addresses ⇒ different objects.

The **class itself occupies no memory** for non-static members.
Memory is allocated only when: objects are created

(see [[01 - Introduction, OOP & Pointers#Stack vs Heap — Two Ways to Create Objects|Stack vs Heap]] for memory details)

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

ptr->var ≡ (\*ptr).var

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
