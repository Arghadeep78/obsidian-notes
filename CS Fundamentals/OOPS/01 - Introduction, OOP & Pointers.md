# Part 1 — Introduction to OOPs
## What Is OOPs? (The Big Picture First)

Imagine you're building a school management system. Without OOPs, you'd write separate variables for every student:
```
string student1_name, student1_dept;
string student2_name, student2_dept;
// ... 500 students = 1000+ lines just for declarations
```

That's unmanageable. OOPs solves this by saying: **"Define what a student looks like once, then stamp out as many as you need."**

> **Object-Oriented Programming (OOPs)** is a way of writing code by modelling real-world entities as **objects** — each object bundles its own data (what it *is*) and functions (what it *can do*) together.

- C++ supports both procedural and OOPs — you choose the style
- Every major language (C++, Java, Python, JS) uses OOPs — learn it once, apply everywhere
- The C++ standard library itself (`vector`, `string`, `map`) is built using OOPs internally
 

	Inside a class, member functions can access data members and other member functions even if they are declared later in the class definition, because the compiler processes the entire class before compiling its members.


---

## The Two Fundamental Concepts
### Class — The Blueprint
Think of a class like an **architect's blueprint for a house**. The blueprint describes what every house will look like — rooms, doors, windows — but the blueprint itself is not a house. It takes no physical space until you actually build one.

- Defines what **data** (variables) and **behaviour** (functions) each object of that type will have
- **Occupies zero memory by itself** — memory is only allocated when you create an object from it

### Object — The Real Thing

An object is a **concrete, physical instance** of a class — an actual house built from the blueprint.

- Each object gets its **own independent copy** of the data members in memory
- You can create **millions of objects** from the same class — each is independent
- Modifying one object's data does NOT affect another

Class Member are of **2** types:
	Functions associated with a class are called **methods** of that class (aka member functions).
	Class **properties/attributes** are variables or data members defined within a class. (aka data member).

```cpp
#include <iostream>
using namespace std;

class Student {
  private:
	string name;
  public:
    // Inside class definition
    void setName(string n) {
        name = n;
    }
    void display();// Declaration only
}; //semi-colon must
void Student::display() { // Outside class definition
    cout << name << endl;
}
int main() {
    Student s;
    s.setName("Arghadeep");
    s.display();
    cout<<s.name<<endl; //s.name used as var
    return 0;
}
```

---

## Why OOPs? — The "Don't Repeat Yourself" Argument

**Without OOPs:** Every teacher needs separate variables. Adding a new field means editing 100+ lines.
```cpp
string t1_name, t1_dept;   // Teacher 1's data
string t2_name, t2_dept;   // Teacher 2's data — exact same structure, repeated
// For 50 teachers: 100+ declarations. Adding a new field = 50 edits.
```

**With OOPs:** The structure is defined once. Creating a new teacher is one line.
```cpp
Teacher t1, t2, t3;   // all three share the same blueprint
// Need 10,000 teachers? Same code — zero extra declarations.
// Add a new field to the class? One edit, all objects get it automatically.
```

---

## The Four Pillars (MEMORISE)

| Pillar                                                                                                                            | What It Means                                                         | Real-World Analogy                                                                         |
| --------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| **[[02 - Class, Object, Access Specifiers & Encapsulation#Encapsulation — What It Actually Means\|Encapsulation]]**               | Bundle data + methods into one unit (class), and hide sensitive data  | A capsule pill — ingredients inside, outer shell is what you interact with                 |
| **[[06 - Abstraction & Abstract Class#What Is Abstraction? (The ATM Analogy)\|Abstraction]]**                                     | Hide *how* something works internally; expose only *what* it does     | ATM machine — you press "withdraw", you don't see the banking logic                        |
| **[[04 - Inheritance — Modes & Types#What Is Inheritance? (The Intuition First)\|Inheritance]]**                                  | A child class automatically gets properties/methods of a parent class | Child inherits eye colour and height from parents                                          |
| **[[05 - Polymorphism — Overloading, Overriding & Virtual Functions#What Is Polymorphism? (The Intuition First)\|Polymorphism]]** | Same function name behaves differently depending on context           | A person behaves differently as an employee, parent, friend — same person, different roles |

> These four pillars are the **most asked OOPs interview questions**. Know them cold.

---

## Real-World to Code Mapping

| Real World | OOPs Concept | Code Example |
|---|---|---|
| Blueprint of a car | Class | `class Car { ... };` |
| Your specific red Toyota (plate: KA-01) | Object (instance) | `Car myCar;` |
| Colour, speed, fuel level | Data members (properties) | `string colour; int speed;` |
| Accelerate, brake, refuel | Member functions (methods) | `void accelerate() { ... }` |
| Amazon product listing | Object | `Product p; p.name="Phone"; p.price=999;` |
| Building from blueprint | Instantiation | `Car myCar;` — the act of creating an object |

---

## `struct` vs `class` — Two Critical Difference

1. Default member access (`public` vs `private`)
2. Default inheritance access (`public` vs `private`)
3. 
```cpp
struct S {
    int x;   // PUBLIC by default — everyone can read/write x
};
class C {
    int x;   // PRIVATE by default — nobody outside can touch x
};
```

- **Convention:** Use `struct` for simple data containers (just data, no behaviour)
- **Convention:** Use `class` for full OOPs entities with data + functions + access control

---

## Terminology Cheat Sheet

| Term | Plain English Meaning |
|---|---|
| **Class** | Blueprint / template that defines a type |
| **Object / Instance** | A concrete, in-memory realisation of a class |
| **Data Member / Attribute** | A variable stored inside an object |
| **Member Function / Method** | A function defined inside a class that operates on its object |
| **Instantiation** | The act of creating an object from a class (`Teacher t1;`) |
| **`.` operator** | Access members of a stack object (`t1.name`) |
| **`->` operator** | Access members of a pointer to an object (`ptr->name`) |

---

## Interview Q&A

**Q: "What is OOPs?"**
A: A programming paradigm that models real-world entities as objects combining state (data) and behaviour (functions), enabling code reusability, modularity, and maintainability.

**Q: "Name the 4 pillars."**
A: Encapsulation, Abstraction, Inheritance, Polymorphism.

**Q: "What is the difference between class and object?"**
A: A class is a blueprint — defines structure but occupies no memory. An object is an instance — the actual thing in memory created from that blueprint.

**Q: "What is a pointer?"**
A: A variable that stores the memory address of another variable. Declared with `*` (e.g., `int* p`). Dereferenced with `*p` to access the value at that address.

**Q: "What is a dangling pointer?"**
A: A pointer that holds the address of memory that has already been freed. Accessing it is undefined behaviour. Fix: set `ptr = nullptr` after `delete`.

**Q: "What is a memory leak?"**
A: Heap memory allocated with `new` that is never freed with `delete`. Accumulates in long-running programs until the process crashes.

**Q: "`delete` vs `delete[]`?"**
A: `delete` for a single object (`new T`). `delete[]` for arrays (`new T[n]`). Using the wrong one is undefined behaviour.

**Q: "What is `nullptr`?"**
A: A C++11 keyword for a null pointer — safer than `NULL` (which is `0` and causes ambiguity with integer overloads).

**Q: "Stack vs heap?"**
A: Stack: automatic, fast, freed at scope end, limited size. Heap: manual (`new`/`delete`), larger, lives until explicitly freed.

**Q: "What does `p->member` mean?"**
A: Shorthand for `(*p).member` — dereference the pointer to get the object, then access its member.
