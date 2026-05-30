# Part 1 — Inheritance Basics & Access Modes

## What Is Inheritance? (The Intuition First)

Imagine you're modelling a school system. A `Student` has: name, age, rollNo. A `Teacher` has: name, age, subject. A `Staff` has: name, age, employeeId.

Notice: **name** and **age** appear in ALL three. Without inheritance, you'd copy-paste those fields into each class — and if you need to change how `age` is stored, you'd have to edit all three classes.

**Inheritance** solves this: put shared properties in one parent class, and have each child class automatically get them — plus add their own unique stuff.

> **Inheritance** is an OOPs mechanism where a **child (derived) class** automatically acquires the properties and member functions of a **parent (base) class**, enabling code reuse without rewriting.

```
Real world: Children inherit traits (eye colour, height) from parents
OOPs:        Derived class inherits data members and functions from base class
```

**Vocabulary:**

| Term | Synonyms |
|---|---|
| Base class | Parent class, Super class |
| Derived class | Child class, Sub class |
| Relationship | "is-a" (Dog IS-A Animal, Student IS-A Person) |

---

## Why Inheritance? — The DRY Principle

```cpp
// WITHOUT inheritance — name and age repeated in every class
class Student { string name; int age; int rollNo; ... };
class Teacher  { string name; int age; string dept; ... };
class Staff    { string name; int age; int empId; ... };
// If 'age' needs to be 'dateOfBirth' instead → edit 3 files

// WITH inheritance — name and age defined ONCE in Person
class Person   { public: string name; int age; };
class Student  : public Person { public: int rollNo; };
class Teacher  : public Person { public: string dept; };
class Staff    : public Person { public: int empId; };
// If 'age' needs to change → edit only Person
```

---

## Basic Syntax

The `: public ClassName` part says "this class inherits from ClassName":

```cpp
class Base {
    public:
        string name;
        int age;
        Base() { cout << "Parent constructor called\n"; }
};

class Derived : public Base {   // ← "Derived inherits from Base"
    public:
        int rollNo;
        Derived() { cout << "Child constructor called\n"; }
};

int main() {
    Derived d1;         // Creating a Derived object ALSO creates the Base part
    // Output:
    //   Parent constructor called  ← base runs first
    //   Child constructor called   ← then child

    d1.name   = "Rahul";  // ✅ inherited from Base — works directly
    d1.age    = 21;       // ✅ inherited from Base
    d1.rollNo = 1234;     // own property
}
```

---

## Constructor & Destructor Call Order — Important!

When you create a derived object, the **base constructor runs first**, then the derived constructor. When the object is destroyed, the **derived destructor runs first**, then the base destructor — exactly reversed. See [[03 - Constructors, this Pointer, Copy Constructor, Shallow & Deep Copy & Destructor#Constructor — The "Setup Man"|constructors]] and [[03 - Constructors, this Pointer, Copy Constructor, Shallow & Deep Copy & Destructor#Destructor — The "Cleanup Man"|destructors]] for details.

```
Creation:    Base ctor → Derived ctor
Destruction: Derived dtor → Base dtor   (reversed!)
```

```cpp
class Person {
    public:
        Person()  { cout << "Person constructor\n"; }
        ~Person() { cout << "Person destructor\n"; }
};

class Student : public Person {
    public:
        Student()  { cout << "Student constructor\n"; }
        ~Student() { cout << "Student destructor\n"; }
};

int main() {
    Student s1;
}

// Output:
// Person constructor     ← base runs first
// Student constructor
// Student destructor     ← on destruction, child runs first
// Person destructor      ← base runs last
```

**Why this order?** The derived object *contains* the base sub-object inside it. Base must be set up before derived can use it. On destruction, derived must be cleaned up before the base it depends on is torn down.

---

## Passing Values to the Parent Constructor

When the parent has a parameterized constructor, you must explicitly call it from the child's constructor using an **initializer list**:

```cpp
class Person {
    public:
        string name;
        int age;
        Person(string name, int age) {
            this->name = name;
            this->age  = age;
        }
};

class Student : public Person {
    public:
        int rollNo;

        // ": Person(name, age)" — this calls the parent constructor explicitly
        // It runs BEFORE the { body } of Student's constructor
        Student(string name, int age, int rollNo)
            : Person(name, age) {       // ← pass values up to parent
            this->rollNo = rollNo;
        }

        void getInfo() {
            cout << name << " | " << age << " | " << rollNo << endl;
        }
};

int main() {
    Student s1("Rahul Kumar", 21, 1234);
    s1.getInfo();   // Rahul Kumar | 21 | 1234
}
```

---

## Mode of Inheritance — How Access Levels Change

When a class inherits, the **mode** (public/protected/private) controls how the inherited members are re-exposed in the derived class.

> **Golden Rule:** `private` members of the base class are **inherited** (they exist inside the derived object's memory and contribute to its size), but they are **inaccessible** to derived class code regardless of inheritance mode. The memory is there — the compiler just forbids you from naming it directly.

### The Mode Effects Table

| Base member visibility | `public` inheritance | `protected` inheritance | `private` inheritance |
|---|---|---|---|
| `public` member | stays `public` | becomes `protected` | becomes `private` |
| `protected` member | stays `protected` | stays `protected` | becomes `private` |
| `private` member | ❌ inaccessible (but EXISTS in memory) | ❌ inaccessible (but EXISTS in memory) | ❌ inaccessible (but EXISTS in memory) |

> **Important:** "inaccessible" means derived class *code* cannot name or use private base members directly. It does NOT mean those members are absent. They are physically present inside every derived object's memory and count toward its size — the compiler just forbids direct access to them. Use the base class's public/protected methods to interact with them.

```cpp
class Base {
    public:    int x;   // public
    protected: int y;   // protected
    private:   int z;   // private — NEVER accessible to derived class
};

class D1 : public    Base { /* x=public,    y=protected  — interface preserved */ };
class D2 : protected Base { /* x=protected, y=protected  — public things hidden from outside */ };
class D3 : private   Base { /* x=private,   y=private   — everything hidden outside D3 */ };
```

**When to use which:**
- **`public`** — most common; keeps base's interface intact for the world
- **`protected`** — hide from the outside, but share with grandchildren
- **`private`** — completely absorb base's implementation; hide everything

---

# Part 2 — Types of Inheritance & Advanced Topics

## Types of Inheritance

### 1. Single Inheritance — Simplest Form

One parent, one child. The most common type.

```cpp
class Person  { public: string name; int age; };
class Student : public Person { public: int rollNo; };
```

```
Person ──→ Student
```

---

### 2. Multi-Level Inheritance — Chain

A chain of inheritance: A → B → C. Each level inherits from the one above.

```cpp
class Person          { public: string name; int age; };
class Student         : public Person  { public: int rollNo; };
class GradStudent     : public Student { public: string researchArea; };

int main() {
    GradStudent gs;
    gs.name         = "Tony Stark";      // inherited from Person (two levels up)
    gs.rollNo       = 1001;              // inherited from Student (one level up)
    gs.researchArea = "Quantum Physics"; // own property
}
```

```
Person ──→ Student ──→ GradStudent
```

---

### 3. Multiple Inheritance — Two Parents

One child inherits from **two different** parent classes simultaneously.

```cpp
class StudentBase {
    public: string name; int rollNo;
};

class TeacherBase {
    public: string subject; double salary;
};

// TA (Teaching Assistant) is both a student AND a teacher
class TA : public StudentBase, public TeacherBase {
    // inherits all members from both parents
};

int main() {
    TA ta;
    ta.name    = "Tony Stark";    // from StudentBase
    ta.subject = "Engineering";  // from TeacherBase
}
```

```
StudentBase ──┐
              ├──→ TA
TeacherBase  ──┘
```

---

### 4. Hierarchical Inheritance — One Parent, Many Children

One parent class, multiple child classes each inheriting from it.

```cpp
class Person  { public: string name; int age; };

class Student : public Person { public: int rollNo; };
class Teacher : public Person { public: string subject; };
class Staff   : public Person { public: int empId; };
```

```
             Person
           /    |    \
    Student  Teacher  Staff
```

---

### 5. Hybrid Inheritance — A Mix

Combines two or more **different types** of inheritance in the same hierarchy. The example below mixes hierarchical inheritance (Person→Student, Person→Teacher) with multiple inheritance (TA inherits both):

```cpp
class Person    { public: string name; int age; };
class Student   : public Person { public: int rollNo; };
class Teacher   : public Person { public: string subject; };
class TA        : public Student, public Teacher { };  // multiple + hierarchical = hybrid
```

```
        Person
       /      \
  Student    Teacher
       \      /
          TA
```

This specific shape — two parents sharing a common grandparent — is the **Diamond Problem** (covered next). The hybrid aspect is the combination of hierarchical (two classes inherit Person) and multiple (TA inherits both) inheritance types.

---

## Diamond Problem & Virtual Inheritance

The **Diamond Problem** occurs in multiple/hybrid inheritance when two parents share a common base class. The grandchild then has **two copies** of the grandparent's data — causing ambiguity.

```
       Animal
      /       \
   Lion        Tiger
      \       /
        Liger      ← inherits Animal TWICE — which Animal::age is it?
```

```cpp
class Animal {
public:
    int age;
    void eat() { cout << "Eating\n"; }
};

class Lion  : public Animal { };
class Tiger : public Animal { };

class Liger : public Lion, public Tiger { };

int main() {
    Liger l;
    l.age  = 5;    // ERROR: ambiguous — is this Lion::Animal::age or Tiger::Animal::age?
    l.eat();       // ERROR: ambiguous — which Animal::eat()?
}
```

### Fix — `virtual` Inheritance

Add `virtual` when inheriting from the shared base. This tells the compiler: "share ONE copy of Animal, don't duplicate it."

```cpp
class Animal { public: int age; };

class Lion  : virtual public Animal { };   // virtual keyword here
class Tiger : virtual public Animal { };   // and here

class Liger : public Lion, public Tiger { };

int main() {
    Liger l;
    l.age = 5;   // ✅ no ambiguity — single shared Animal instance
    l.eat();     // ✅ one Animal::eat()
}

// Constructor order with virtual inheritance:
// Animal ← virtual base always constructed FIRST (only once)
// Lion
// Tiger
// Liger
```

**Critical interview trap — who calls the virtual base constructor?**

With virtual inheritance, the **most-derived class** (`Liger`) is responsible for directly calling `Animal`'s constructor — NOT `Lion` or `Tiger`. The Lion/Tiger initializer-list calls to `Animal(...)` are silently **ignored** when constructing a `Liger`:

```cpp
class Animal {
public:
    int age;
    Animal(int a) : age(a) { cout << "Animal(" << a << ")\n"; }
};

class Lion  : virtual public Animal {
public:
    Lion()  : Animal(0) { }   // this call is IGNORED when Liger is constructed
};
class Tiger : virtual public Animal {
public:
    Tiger() : Animal(0) { }   // this call is also IGNORED when Liger is constructed
};

class Liger : public Lion, public Tiger {
public:
    Liger() : Animal(10), Lion(), Tiger() { }
    //         ↑ Liger must call Animal() directly — this is the one that actually runs
};
```

If `Liger` doesn't explicitly call `Animal(...)` in its initializer list, the compiler calls `Animal`'s **default constructor**. Forgetting this when `Animal` only has a parameterized constructor is a compile error.

**Cost of virtual inheritance — know this for interviews:**
Virtual inheritance adds a hidden pointer (the *virtual base pointer*, sometimes called `vbptr`) to the layout of intermediate classes (`Lion`, `Tiger`) so they can locate the single shared `Animal` sub-object at runtime. This means:
- Each `Lion`/`Tiger` object is slightly larger than without `virtual`
- Construction order is more complex (virtual base always initialized first by the most-derived class)
- A small runtime cost exists when accessing the virtual base's members through an intermediate pointer

This is why you should **not** use `virtual` inheritance by default — only reach for it when you actually have the diamond problem. Prefer composition or interface-only (all-pure-virtual) bases to avoid the need for it entirely.

---

## Multiple Inheritance Name Collision

Even without the diamond problem, two parents with a **same-named function** cause ambiguity:

```cpp
class A { public: void show() { cout << "A::show\n"; } };
class B { public: void show() { cout << "B::show\n"; } };
class C : public A, public B { };

int main() {
    C obj;
    obj.show();       // ERROR: ambiguous — A::show or B::show?

    obj.A::show();    // ✅ explicit scope resolution — picks A's version
    obj.B::show();    // ✅ explicit scope resolution — picks B's version
}

// Or: define show() in C itself to override both:
class C : public A, public B {
public:
    void show() { A::show(); cout << "C::show\n"; }   // now unambiguous
};
```

---

## Object Slicing — A Hidden Danger

When a derived object is assigned **by value** to a base class variable, the derived-only members are *sliced off* and lost:

```cpp
Student s;   s.name = "Rahul"; s.rollNo = 1;
Person p = s;   // SLICING — p is a Person, so rollNo is lost
// p.rollNo doesn't exist — only name and age remain

// Prevention: use pointers or references (see [[05 - Polymorphism — Overloading, Overriding & Virtual Functions#Virtual Functions — The Core of Runtime Polymorphism|virtual functions]] for polymorphic use)
Person* ptr = &s;   // ✅ no slicing — ptr points to full Student object
Person& ref = s;    // ✅ no slicing — ref refers to full Student object
```

---

## `protected` Members in Practice

```cpp
class Animal {
    protected:
        string sound;   // accessible inside Animal AND its children, NOT from outside
};

class Dog : public Animal {
    void bark() { sound = "Woof"; }   // ✅ legal — Dog is a child of Animal
};

// Dog d; d.sound = "X";  ← ❌ illegal — outside the class hierarchy
```

---

## Interview Q&A

**Q: "What is inheritance?"**
A: A mechanism where a derived class automatically acquires properties and methods of a base class, enabling code reuse and establishing "is-a" relationships.

**Q: "What is the constructor call order in inheritance?"**
A: Base constructor first, then derived. Destructors run in reverse: derived first, base last.

**Q: "Difference between multi-level and multiple inheritance?"**
A: Multi-level: A→B→C (a chain — B inherits A, C inherits B). Multiple: A+B→C (two parents — C inherits from both A and B).

**Q: "What is the diamond problem?"**
A: When a class inherits from two classes that both inherit from a common base, the grandchild gets two separate copies of the grandparent, causing ambiguity. Fixed with `virtual` inheritance.

**Q: "What are the three modes of inheritance?"**
A: `public` — preserves base's public/protected access intact. `protected` — makes public become protected. `private` — makes both public and protected become private. Private base members are never accessible regardless of mode.

**Q: "What is object slicing?"**
A: When a derived object is assigned by value to a base variable, the derived-specific members are lost. Prevent by using pointers or references instead.
