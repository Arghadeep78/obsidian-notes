# Part 1 — Inheritance Basics & Access Modes

## What Is Inheritance?

A child inherits traits from their parents. For example, a child may inherit the family surname, eye color, or certain characteristics from their parents while also having their own unique qualities.

A `Student`, `Teacher`, and `Staff` all have `name` and `age`. Instead of duplicating these fields in every class, *inheritance* lets you place shared data in a parent class and have child classes reuse it while adding their own unique members (`rollNo`, `subject`, `employeeId`).

**Inheritance** solves this: put shared properties in one parent class, and have each child class automatically get them — plus add their own unique stuff.

> **Inheritance** is an OOPs mechanism where a **child (derived) class** automatically acquires the properties and member functions of a parent (**base**) **class**, enabling **code reuse** without rewriting.

```css
Real world:  Children inherit traits from parents
OOPs:        Derived class inherits data members and functions from base class
```

**Vocabulary:**

| Term          | Synonyms                                      |
| ------------- | --------------------------------------------- |
| Base class    | Parent class, Super class                     |
| Derived class | Child class, Sub class                        |

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
class BaseClass {
    public:
        string name;
        int age;
        Base() { cout << "Parent constructor called\n"; }
};

class DerivedClass : public Base {   // ← "Derived inherits from Base"
    public:
        int rollNo;
        Derived() { cout << "Child constructor called\n"; }
};

int main() {
    DerivedClass d1;         // Creating a Derived object ALSO creates the Base part
    // Output:
    //   Parent constructor called  ← base runs first
    //   Child constructor called   ← then child

    d1.name   = "Rahul";  // ✅ inherited from Base — works directly
    d1.age    = 21;       // ✅ inherited from Base
    d1.rollNo = 1234;     // own property
}
```

---

## Constructor & Destructor Call Order

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

When the parent has a parameterized constructor, you must explicitly call it from the child's constructor using an [[03 - Constructors, this Pointer, Copy Constructor, Shallow & Deep Copy & Destructor#3 ways to write a constructor | Initializer List]]

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
```
#### Syntax:
```cpp
class Student : public Person {
    public:
        int rollNo;
        // ": Person(name, age)" — this calls the parent constructor explicitly
        // It runs BEFORE the { body } of Student's constructor
        Student(string name, int age, int rollNo) : Person(name, age) {       // ← pass values up to parent
            this->rollNo = rollNo;
        }
};
```

---

## Mode of Inheritance — How Access Levels Change

When a class inherits, the **mode** (public/protected/private) controls how the inherited members are re-exposed in the derived *class*.

> **Golden Rule:** `private` members are **inherited** and occupy memory in the derived object, but the derived class **cannot access** them *directly*. Access is only possible through the base class's public/protected functions (e.g., getters and *setters*).

### The Mode Effects Table

| **Base** /                 *Derived* | *public*                              | *protected*                           | *private*                             |
| ------------------------------------ | ------------------------------------- | ------------------------------------- | ------------------------------------- |
| **public**                           | public                                | protected                             | private                               |
| **protected**`                       | protected                             | protected                             | private                               |
| **private**                          | ❌ inaccessible (but EXISTS in memory) | ❌ inaccessible (but EXISTS in memory) | ❌ inaccessible (but EXISTS in memory) |


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
```

```css
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
```

```css
StudentBase  ──┐
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

This specific shape — two parents sharing a common grandparent — is the **Diamond Problem** (covered next). The hybrid aspect is the combination of *hierarchical* and *multiple* inheritance types.

---

## Scope Resolution Operator` ::`

The `::` operator is used to specify the scope to which a variable, function, class member, or namespace member belongs.

**Uses:**
- Define class member functions outside the class.
- Access global variables hidden by local variables.
- Access static class members.
- Access namespace members.

**Example:**

```
ClassName::functionName()
::Name = "Arghadeep" //global scope
```

**In short:** `::` tells the compiler where to look for a name. avoids conflicts..

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
    l.age  = 5;    // ERROR: ambiguous — Lion::Animal::age or Tiger::Animal::age?
    l.eat();       // ERROR: ambiguous — which Animal::eat()?
}
```

### Fix — `virtual` Inheritance

Add `virtual` when inheriting from the shared base. This tells the compiler: "share ONE copy of Animal, don't duplicate it."

```cpp
class Animal { public: int age; void eat() { cout << "Eating\n"; } };

class Lion  : virtual public Animal { };   // virtual keyword here
class Tiger : virtual public Animal { };   // and here

class Liger : public Lion, public Tiger { };

int main() {
    Liger l;
    l.age = 5;   // ✅ no ambiguity — single shared Animal instance
    l.eat();     // ✅ one Animal::eat()
}
```
#### Constructor order with virtual inheritance:

``` Animal -> Lion -> Tiger > Liger ```

Virtual base classes(Animal) are always constructed first and only once.

```
						Animal
						   |
						(shared)
						 /      \
						Lion   Tiger
						 \      /
						   Liger
```

**Critical interview trap — who calls the virtual base constructor?**

When `Liger l;` is executed, the constructors run in this exact order:
1. `Animal(10)` runs first (direct initialization from the most derived class).
2. `Lion()` runs next (initializes the `Lion` specific fields; its internal call to `Animal` is ignored).
3. `Tiger()` runs last (initializes the `Tiger` specific fields; its internal call to `Animal` is ignored).
As a result, `l.id` is set to `10`.

```cpp
class Animal {  
public:  
	int id;
	Animal(int x) {this->id = x;}  
};  
  
class Lion : virtual public Animal {  
public:  
	Lion() : Animal(1) {} // ignored when Liger Constructed 
};  
  
class Tiger : virtual public Animal {  
public:  
	Tiger() : Animal(2) {} // ignored when Liger Constructed 
};  
  
class Liger : public Lion, public Tiger {  
public:  
	Liger() : Animal(10) {} // actually runs  
	//without virtual error: `Animal` is not a direct base of `Liger`
	//after this compiler calls Lion() and Tiger() by default
};
```

If `Liger` doesn't explicitly call `Animal(...)` in its initializer list, the compiler calls `Animal`'s **default constructor**. Forgetting this when `Animal` only has a parameterized constructor is a compile error.

#### Note
- ``` Liger() : Animal(10) {} is equivalent to Liger() : Animal(10), Lion(), Tiger() {} ```
	these are default constructor of Lion and Tiger (must be defined if parameterized constructor defined)
- C++ ignores order of initializer list so Lion() is always called before Tiger() even if Animal(10), Tiger(), Lion()
#### Rule:
- Non-virtual inheritance → immediate parent constructs the base.
- Virtual inheritance → most-derived class constructs the virtual base.

**Cost of Virtual Inheritance**

- Adds a hidden pointer (`vbptr`, *virtual base pointer*), to layout of intermediate classes(to locate shared sub-object(*Animal*) at runtime)
- Increases object size of intermediate classes.
- Slightly more complex construction (virtual base initialized first by the most-derived class).
- Small runtime overhead when accessing the virtual base (intermediate pointer).

Use virtual inheritance *only to solve the diamond problem*; otherwise avoid the extra complexity and overhead.

#### Ways to Avoid Virtual Inheritance

Prefer **composition** or **interfaces** (*pure virtual function.*) to avoid the need for it entirely.

##### interfaces
An **interface** defines a contract—a set of actions an object must support—without providing the implementation details.
	An interface is typically an abstract class containing [[06 - Abstraction & Abstract Class#Way 2 — Abstraction via Abstract Class|Pure Virtual Functions]] (and usually a virtual destructor) and no data members. Check out [[06 - Abstraction & Abstract Class#Interface vs Abstract Class (C++ Perspective)|Interfaces]]
```cpp
class Animal {  
public:  
	virtual void speak() = 0;  // pure virtual function.
}; 
class Lion : public Animal {  
public:  
	void speak() override {  
		cout << "Roar\n";  
	}  
};   
class Tiger : public Animal { public:  void speak() override {  cout << "Growl\n";  } };  
class Liger : public Lion, public Tiger {  
public:  
	void speak() override {  
		Lion::speak(); // choose Lion's version   (must be overidden otherwise compiler don't who which version to use -> error)
	}  
};  
int main() {  
	Liger l;  
	l.speak(); // Roar  
}
```
##### composition
**Composition** is a design principle where a complex object is built by combining one or more simpler objects. Instead of inheriting behavior from a parent class (**Is-A**), a class holds instances of other classes as fields (**Has-A**).
```cpp
	class Car : public Engine { };
	//vs
	class Car {
	    Engine engine;   // Car HAS-A Engine
	};
```

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

## Object Slicing

When a derived object is assigned **by value** to a base class variable, the derived-specific members and [[vtable]] pointer are lost are *sliced off* and lost:
``` BMW b;  Car c = b; // slicing ```

A **new `Animal` object** is created, so only the `Animal` part of `d` is copied. All `Dog`-specific data is discarded.

`d` points to a `Dog` object, but its **compile-time type** is `Car*`
```cpp
Dog d; d.name = "Bruno";
Animal a = d;    // SLICING — only Animal part copied, Dog-ness is lost
a.speak();       // calls Animal::speak() — virtual dispatch is gone after slicing!

// Prevention: always use pointers or references for polymorphic objects
Animal* ptr = &d;   // ✅ no slicing — full Dog object alive
Animal& ref = d;  //alis to d  : ✅ no slicing — (another name) for `d`
```

To call *Dog-specific functions*, you must cast:
``` cpp
	Dog* dogPtr = dynamic_cast<Dog*>(ptr); //dynamic returns nullptr if cast is invalid
```

---

## `protected` Members in Practice

```cpp
class Animal {
    protected:
        string bark;   // accessible inside Animal AND its children, NOT from outside
};

class Dog : public Animal {
    void setBark() { bark = "Woof"; }   // ✅ legal — Dog is a child of Animal
};

Dog d;
d.bark = "woof woof";  ← ❌ illegal — outside the class hierarchy
```

`protected` = accessible inside the class and its derived classes(public, protected, private all modes, but not from outside.

With *private inheritance*, base class public/protected members become private in the derived class, so further derived classes cannot access them.

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
