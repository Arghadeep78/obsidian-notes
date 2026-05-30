# Part 1 — Compile-Time Polymorphism

## What Is Polymorphism? (The Intuition First)

Consider a real person — you *behave* differently as a student in class, as a son/daughter at home, and as a friend in a group. Same person, same name, different behaviour depending on context.

In code: same function name, different behaviour depending on the context (arguments, or which object calls it).

> **Poly** = many | **Morph** = forms
> **Polymorphism** = the ability of a function or object to behave differently depending on context, even when called by the same name.

---

## Two Types — Compile-Time vs Run-Time

```
Polymorphism
├── Compile-Time (Static Polymorphism)
│   Resolved at COMPILE TIME — the compiler knows which version to call
│   ├── Function Overloading
│   ├── Constructor Overloading (see [[03 - Constructors, this Pointer, Copy Constructor, Shallow & Deep Copy & Destructor#Constructor Overloading|constructor overloading]])
│   └── Operator Overloading
│
└── Run-Time (Dynamic Polymorphism)
    Resolved at RUN TIME — decided only when the program is actually executing
    ├── Function Overriding (requires [[04 - Inheritance — Modes & Types#What Is Inheritance? (The Intuition First)|inheritance]])
    └── Virtual Functions (the mechanism that makes it work)
```

---

## Compile-Time Polymorphism

### Function Overloading

> **Definition:** Multiple functions in the **same class** with the **same name** but **different parameters** (different type, or different count).

The compiler looks at the *argument types* at compile time and picks the right version — zero runtime cost.

```cpp
#include <iostream>
using namespace std;

class Printer {
    public:
        void show(int x) {           // version 1: takes an int
            cout << "Integer: " << x << endl;
        }

        void show(char c) {          // version 2: takes a char
            cout << "Character: " << c << endl;
        }

        void show(double d) {        // version 3: takes a double
            cout << "Double: " << d << endl;
        }

        void show(int x, int y) {    // version 4: takes two ints
            cout << "Sum: " << x + y << endl;
        }
        // Same name "show" — 4 different behaviours depending on arguments
};

int main() {
    Printer p;
    p.show(10);      // compiler picks version 1 → Integer: 10
    p.show('A');     // compiler picks version 2 → Character: A
    p.show(3.14);    // compiler picks version 3 → Double: 3.14
    p.show(3, 4);    // compiler picks version 4 → Sum: 7
}
```

**Rules for function overloading:**
- Must differ in **type** or **count** of parameters
- Return type alone is NOT enough — `void show()` and `int show()` is a compile error
- The compiler resolves it at compile time → no runtime overhead

---

### Operator Overloading

Extend what built-in operators (`+`, `==`, `<<`) do for your custom types:

```cpp
class Complex {
    public:
        double real, imag;
        Complex(double r, double i) : real(r), imag(i) {}

        // Overload + so Complex + Complex works naturally
        Complex operator+(const Complex& other) const {
            return Complex(real + other.real, imag + other.imag);
        }

        void print() {
            cout << real << " + " << imag << "i" << endl;
        }
};

int main() {
    Complex c1(2, 3), c2(1, 4);
    Complex c3 = c1 + c2;   // calls operator+() — same as c1.operator+(c2)
    c3.print();              // 3 + 7i
}
```

---

# Part 2 — Run-Time Polymorphism & Virtual Functions

## Run-Time Polymorphism

### Function Overriding

> **Definition:** A **child class** redefines a function from the **parent class** with the **exact same name and signature** but a **different implementation**.

Unlike overloading, overriding requires **inheritance**. The child's version *replaces* (overrides) the parent's version for that child type.

```cpp
#include <iostream>
using namespace std;

class Animal {
    public:
        void speak() {
            cout << "Some animal sound\n";   // generic default
        }
};

class Dog : public Animal {
    public:
        void speak() {                        // SAME name, SAME signature — overrides parent's version
            cout << "Woof!\n";               // Dog-specific behaviour
        }
};

class Cat : public Animal {
    public:
        void speak() {
            cout << "Meow!\n";
        }
};

int main() {
    Animal a;  a.speak();   // "Some animal sound" — calls Animal's version
    Dog    d;  d.speak();   // "Woof!" — Dog's version, not Animal's
    Cat    c;  c.speak();   // "Meow!" — Cat's version
}
```

---

## Overloading vs Overriding — Side by Side

| Feature | Overloading | Overriding |
|---|---|---|
| Where? | Same class | Parent class + Child class (needs inheritance) |
| Signature | Different parameters | SAME parameters, same name |
| Resolved at | Compile time | Run time |
| Type | Static polymorphism | Dynamic polymorphism |
| Keyword needed | None | `virtual` (for proper dynamic dispatch) |

---

## Virtual Functions — The Core of Runtime Polymorphism

### The Problem Without `virtual`

Suppose you have a base class pointer pointing to a derived class object. Without `virtual`, calling a function through that pointer always calls the **base class version**, even if the derived class overrode it. This is called **static dispatch** — the compiler picks the function based on the *pointer type*, not the *actual object type*.

```cpp
class Animal {
    public:
        void speak() { cout << "Animal sound\n"; }  // NOT virtual
};

class Dog : public Animal {
    public:
        void speak() { cout << "Woof!\n"; }  // overrides, but without virtual...
};

int main() {
    Animal* ptr = new Dog();   // base pointer → derived object (common in OOPs)
    ptr->speak();              // prints: "Animal sound" ← WRONG!
                               // Compiler sees ptr is Animal* → calls Animal::speak
                               // It doesn't care that the actual object is a Dog
}
```

This is a big problem: if you have a list of `Animal*` pointers (some pointing to Dogs, some to Cats), calling `speak()` would always give you the generic Animal version — useless.

### The Fix — `virtual` Keyword

Adding `virtual` to the base class function tells the compiler: "when this function is called through a pointer/reference, don't decide at compile time — look up the actual object type at **runtime** and call the right version."

```cpp
class Animal {
    public:
        virtual void speak() {           // declare virtual in BASE class
            cout << "Animal sound\n";
        }
};

class Dog : public Animal {
    public:
        void speak() override {          // override keyword (C++11) — optional but good practice
            cout << "Woof!\n";           // compiler verifies this actually overrides something
        }
};

class Cat : public Animal {
    public:
        void speak() override {
            cout << "Meow!\n";
        }
};

int main() {
    Animal* ptr1 = new Dog();   // base pointer → Dog object
    Animal* ptr2 = new Cat();   // base pointer → Cat object

    ptr1->speak();   // "Woof!" ← correct! Runtime dispatch → Dog::speak
    ptr2->speak();   // "Meow!" ← correct! Runtime dispatch → Cat::speak

    delete ptr1; delete ptr2;
}
```

---

## How Virtual Functions Work Internally — vtable

When a class has virtual functions, the compiler creates a hidden **vtable (virtual function table)** for that class — essentially an array of function pointers.

```
Animal vtable:  [ speak → Animal::speak ]
Dog vtable:     [ speak → Dog::speak    ]
Cat vtable:     [ speak → Cat::speak    ]

Each object has a hidden pointer (vptr) to its class's vtable.

Animal* ptr = new Dog();
→ ptr->speak()
→ follow vptr in the Dog object → Dog's vtable → Dog::speak → "Woof!"
```

This lookup happens at **runtime**, which is why it's called dynamic dispatch. The small cost: one extra pointer indirection per virtual call.

---

## The `override` Keyword (C++11)

`override` is optional but strongly recommended. It tells the compiler: "I *intend* this to override a base class function — if it doesn't, give me a compile error."

```cpp
class Animal { virtual void speak() {} };

class Dog : public Animal {
    void speak() override { }    // ✅ — confirmed it overrides Animal::speak

    void Speak() override { }    // ❌ COMPILE ERROR — no 'Speak' (capital S) in Animal
    // Without override, this silently creates a NEW function, not an override — a nasty bug
};
```

---

## The `final` Keyword (C++11)

`final` prevents further overriding or inheritance:

```cpp
class Animal   { virtual void speak() {} };
class Dog      : public Animal { void speak() final {} };  // no class can override speak() past Dog
class Poodle   : public Dog   {
    // void speak() override {} ← ERROR: speak() is final in Dog
};

class Immutable final { };   // no class can inherit from Immutable at all
```

---

## Complete Polymorphism Example — The Power of Runtime Dispatch

This is the pattern you'll use in real software:

```cpp
#include <iostream>
using namespace std;

class Shape {
    public:
        virtual void draw() {        // virtual — expect every shape to override this
            cout << "Drawing a shape" << endl;
        }
        virtual ~Shape() {}          // virtual destructor — always needed when using virtual functions
};

class Circle : public Shape {
    public:
        void draw() override { cout << "Drawing Circle" << endl; }
};

class Rectangle : public Shape {
    public:
        void draw() override { cout << "Drawing Rectangle" << endl; }
};

class Triangle : public Shape {
    public:
        void draw() override { cout << "Drawing Triangle" << endl; }
};

// ONE function handles ALL shape types — you never need to modify render()
// when you add a new shape type
void render(Shape* s) {
    s->draw();   // dynamic dispatch — correct version called based on actual object
}

int main() {
    Shape* shapes[] = { new Circle(), new Rectangle(), new Triangle() };

    for (Shape* s : shapes) {
        render(s);   // draws each correctly without knowing the concrete type
    }

    // Output:
    // Drawing Circle
    // Drawing Rectangle
    // Drawing Triangle

    for (Shape* s : shapes) delete s;
}
```

This is the **power of runtime polymorphism** — `render()` works for any shape you'll ever create, even ones that don't exist yet.

---

## Virtual Destructor — Critical Rule

> **If a class has ANY virtual function, its destructor MUST also be `virtual`.**

Without a virtual destructor, deleting a derived object through a base pointer only calls the base destructor — the derived destructor (and any cleanup it does) is skipped:

```cpp
class Base {
    public:
        ~Base() { cout << "Base dtor\n"; }   // NOT virtual — DANGER
};

class Derived : public Base {
    int* data;
    public:
        Derived() { data = new int[100]; }
        ~Derived() { delete[] data; cout << "Derived dtor\n"; }   // never called!
};

Base* ptr = new Derived();
delete ptr;
// Only "Base dtor" prints — Derived's data[] is LEAKED
```

**Fix:**
```cpp
class Base {
    public:
        virtual ~Base() { cout << "Base dtor\n"; }   // virtual — now safe
};
// delete ptr; → "Derived dtor" then "Base dtor" — correct!
```

---

## Object Slicing — When Polymorphism Breaks

When you assign a derived object **by value** to a base variable, the derived-specific members and vtable pointer are lost. See [[04 - Inheritance — Modes & Types#Object Slicing — A Hidden Danger|object slicing in inheritance]] for details.

```cpp
Dog d; d.name = "Bruno";
Animal a = d;    // SLICING — only Animal part copied, Dog-ness is lost
a.speak();       // calls Animal::speak() — virtual dispatch is gone after slicing!

// Prevention: always use pointers or references for polymorphic objects
Animal* ptr = new Dog();   // ✅ no slicing — full Dog object alive
Animal& ref = d;           // ✅ no slicing — reference to full Dog
```

---

## Summary Table — Overloading vs Overriding vs Virtual

| | Overloading | Overriding (without virtual) | Overriding (with virtual) |
|---|---|---|---|
| Where | Same class | Parent + Child | Parent + Child |
| Same signature? | No (differs) | Yes | Yes |
| Resolved at | Compile time | Compile time (pointer type decides) | Run time (actual object type decides) |
| Base pointer dispatch? | N/A | Always calls **base** version through base pointer (derived version only reachable via derived pointer/reference) | Calls **derived** version — correct runtime dispatch |
| Keyword | None | None | `virtual` in base |

> **Why "compile time" for non-virtual overriding:** without `virtual`, the compiler binds the call to a function based solely on the *static type* of the pointer/reference (i.e., what type the pointer is declared as). If `ptr` is `Animal*`, `ptr->speak()` always calls `Animal::speak` — even if the actual object at runtime is a `Dog`. The dispatch decision is baked in at compile time.

---

## Name Hiding — The Related Trap (Distinct from Overriding)

When a derived class defines a function with the **same name** as a base class function but a **different signature**, it does NOT overload the base class version — it **hides** all base class overloads of that name from the derived class scope. This is different from overriding.

```cpp
class Base {
public:
    void show(int x)    { cout << "Base::show(int)\n"; }
    void show(double x) { cout << "Base::show(double)\n"; }
};

class Derived : public Base {
public:
    void show(string s) { cout << "Derived::show(string)\n"; }
    // Derived defined show() with a NEW signature
    // This HIDES both Base::show(int) and Base::show(double) — they are invisible in Derived's scope
};

int main() {
    Derived d;
    d.show("hello");   // ✅ Derived::show(string)
    d.show(42);        // ❌ COMPILE ERROR — Base::show(int) is hidden!
    d.show(3.14);      // ❌ COMPILE ERROR — Base::show(double) is hidden!

    // Fix: bring base versions back into scope explicitly
    // Add "using Base::show;" inside Derived — unhides all Base::show overloads
}
```

**Why this trips people up:** It looks like overloading (same name, different params) but it works completely differently. The `override` keyword won't help here because you're not overriding anything (different signature). Use `using Base::show;` inside the derived class to restore all base overloads.

| Situation | What Happens |
|---|---|
| Same name, same signature, `virtual` in base | **Overriding** — correct runtime dispatch |
| Same name, same signature, no `virtual` | **Non-virtual shadowing** — derived version called on derived objects; base version called through base pointer (static dispatch). The base version is NOT hidden — it's reachable via `Base::func()` or a base pointer. |
| Same name, different signature, in derived | **Hiding** — ALL base overloads of that name hidden in derived scope; fix with `using Base::show;` |
| Same name, different signature, in same class | **Overloading** — both versions visible and callable |

---

## Interview Q&A

**Q: "What is polymorphism?"**
A: The ability of an entity to take multiple forms — same interface, different behaviour depending on context.

**Q: "Two types of polymorphism?"**
A: Compile-time (static) — resolved at compile time via function/operator overloading. Run-time (dynamic) — resolved at runtime via virtual functions and overriding.

**Q: "What is function overloading?"**
A: Defining multiple functions with the same name in the same class but with different parameters (type or count). The compiler picks the right version at compile time.

**Q: "What is function overriding?"**
A: Redefining a base class function in a derived class with the exact same signature. Requires inheritance. Shows runtime polymorphism when used with virtual functions.

**Q: "What is a virtual function?"**
A: A member function declared `virtual` in the base class, enabling dynamic dispatch — the actual derived class version is called at runtime even when accessed through a base class pointer.

**Q: "What happens without `virtual` when using base pointer?"**
A: The base class version is always called (static dispatch) — even if the object is actually a derived type. This is wrong behaviour in most polymorphic designs.

**Q: "Why use `virtual` destructor?"**
A: To ensure the derived class destructor runs when deleting through a base pointer. Without it, only the base destructor runs → derived resources leak.

**Q: "What is the `override` keyword?"**
A: A C++11 keyword placed after a function declaration to tell the compiler "this is intended as an override." If the base class has no matching virtual function, the compiler gives an error — catching typos and signature mismatches.

---

## Special Topic — Covariant Return Types

**Q: "Can an overriding function return a different type than the base class virtual function?"**

The answer is **yes — but only if the return type is a derived class of the base function's return type**. This is called a **covariant return type**.

```cpp
class Animal {
public:
    virtual Animal* clone() {      // base returns Animal*
        return new Animal();
    }
};

class Dog : public Animal {
public:
    Dog* clone() override {        // override returns Dog* — a MORE SPECIFIC type
        return new Dog();          // Dog* is covariant with Animal* (Dog IS-A Animal)
    }
};

int main() {
    Dog d;
    Dog* d2 = d.clone();    // ✅ returns Dog* directly — no cast needed
    delete d2;

    Animal* a = &d;
    Animal* a2 = a->clone();  // ✅ also works — virtual dispatch calls Dog::clone
    delete a2;
}
```

**Why it's useful:** Without covariant return types, `Dog::clone()` would have to return `Animal*`, and callers who *know* they have a `Dog` would need to `static_cast` the result. Covariant return types eliminate that cast.

**The rule:** The overriding function's return type must be a pointer (or reference) to a class that is derived from the base function's return type. Raw values (e.g., returning `Dog` instead of `Animal`) don't qualify — only pointers and references.
