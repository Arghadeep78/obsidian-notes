# Part 1 — Compile-Time Polymorphism

## What Is Polymorphism?



> **Poly** = many | **Morph** = forms
> 
> **Polymorphism** = the ability of a function or object to behave differently depending on context in which they are used, even when called by the same name.

---

## Two Types — Compile-Time vs Run-Time

Polymorphism
- Compile-Time (Static Polymorphism)
   Resolved at COMPILE TIME — the compiler knows which version to call
	- Function Overloading
	- Constructor Overloading (see [[03 - Constructors, this Pointer, Copy Constructor, Shallow & Deep Copy & Destructor#Constructor Overloading|constructor overloading]])
	- Operator Overloading

-  Run-Time (Dynamic Polymorphism)
    Resolved at RUN TIME — decided only when the program is actually executing
    ├── Function *Overriding* (requires [[04 - Inheritance — Modes & Types#What Is Inheritance? (The Intuition First)|inheritance]])
    └── Virtual Functions (the mechanism that makes it work)

---

## Compile-Time Polymorphism

### Function Overloading

> **Definition:** Multiple functions in the **same class** with the **same name** but **different parameters** (different type, or different count).

The compiler looks at the *argument types* at compile time and picks the right version — zero runtime cost.

```cpp
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

For example the string library in c++ overloads the = operator to allow us to do s1=s2

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

### Function Overriding & Virtual Functions

> **Definition:** A **child class** redefines a function from the **parent class** with the **exact same name and signature** but a **different implementation**.

Unlike overloading, overriding requires **inheritance**. The child's version *replaces* (overrides) the parent's version for that child type.

True overriding requires `virtual functions`.

#### The Problem Without `virtual` (Static Dispatch)

When a base class pointer points to a derived class object, calling a non-virtual function executes the base class version. The compiler selects the function based on the **pointer's type**, not the *actual object type.*

This is a big problem: if you have a list of `Animal*` pointers (some pointing to Dogs, some to Cats), calling `speak()` would always give you the generic Animal version — useless.

#### The Fix — `virtual` Keyword (Dynamic Dispatch)

Adding the `virtual` keyword to the base class function defers the function call resolution to runtime. The application selects the function based on the **actual object type**.

> A **virtual function** is a member function that we expect to be redefined in the derived class.
> 
- **Dynamic Nature:** Resolved at runtime rather than compile time.
- **Overriding:** Redefined and implemented by a child (derived) class to provide specific behavior.

```cpp
class Animal {
public:
    // Without virtual -> Static dispatch
    void walk() { cout << "Animal walks\n"; }
    // With virtual -> Dynamic dispatch
    virtual void speak() { cout << "Animal sound\n"; }
    virtual ~Animal() = default; //always needed to prevent memory leak
};
class Dog : public Animal {
public:
    void walk() { cout << "Dog runs\n"; }
    void speak() override { cout << "Woof!\n"; } // override keyword (C++11) — optional but good practice
};
int main() {
    Animal* ptr = new Dog();
    // === Calls Base Class Versions (Static Dispatch) ===
    ptr->walk();  // Prints: "Animal walks"
    ptr->speak(); // Prints: "Woof!"
    delete ptr;
}
```

#### Why we need to explicitly state virtual destructors
They are only mandatory when you delete a derived class object through a base class pointer.
The compiler only looks at the pointer type (`Animal*`).
So rule of thumb: If a class has ANY virtual function, its destructor MUST also be `virtual`.
as we intend to work with those.

```cpp
delete ptr;
   │
   └──► Only calls ~Animal()
        (Memory allocated by Dog is leaked)
```
first derived class destructor runs then derived class destructor automatically triggers the base class destructor.

---
## Overloading vs Overriding — Side by Side

| Feature        | Overloading          | Overriding                                     |
| -------------- | -------------------- | ---------------------------------------------- |
| Where?         | Same class           | Parent class + Child class (needs inheritance) |
| Signature      | Different parameters | SAME parameters, same name                     |
| Resolved at    | Compile time         | Run time                                       |
| Type           | Static polymorphism  | Dynamic polymorphism                           |
| Keyword needed | None                 | `virtual` (for proper dynamic dispatch)        |

---

## How Virtual Functions Work Internally — vtable

When a class has virtual functions, the compiler creates a hidden **vtable (virtual function table)** for that class — essentially an array of function pointers.

```
Animal vtable:  [ speak → Animal::speak ]
Dog vtable:     [ speak → Dog::speak    ]
Cat vtable:     [ speak → Cat::speak    ]
```
Each object has a hidden pointer (*vptr*) to its *class's* *vtable*.
```
Animal* ptr = new Dog();
→ ptr->speak()
→ follow vptr in the Dog object → Dog's vtable → Dog::speak → "Woof!"
```

This lookup happens at **runtime**, which is why it's called dynamic dispatch. The small cost: one extra pointer indirection per virtual call.

---

## The `override` Keyword (C++11)

`override` is optional but strongly recommended. It tells the compiler: "I *intend* this to override a base class function — if it doesn't, give me a compile error."

**With it:** Forces compile-time checks. The compiler throws an error if the function does not exactly match a base virtual function.

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
        virtual ~Shape() {}          // virtual destructor — mandatory when deleting a derived class object through a base class pointer.
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

This is the **power of runtime polymorphism** — `render()` works for any shape you'll ever create.

---

## Summary Table — Overloading vs Overriding vs Virtual

|                        | Overloading  | Overriding (without virtual)                                                                                      | Overriding (with virtual)                            |
| ---------------------- | ------------ | ----------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------- |
| Where                  | Same class   | Parent + Child                                                                                                    | Parent + Child                                       |
| Same signature?        | No (differs) | Yes                                                                                                               | Yes                                                  |
| Resolved at            | Compile time | Compile time (pointer type decides)                                                                               | Run time (actual object type decides)                |
| Base pointer dispatch? | N/A          | Always calls **base** version through base pointer (derived version only reachable via derived pointer/reference) | Calls **derived** version — correct runtime dispatch |
| Keyword                | None         | None                                                                                                              | `virtual` in base                                    |
|                        |              |                                                                                                                   |                                                      |

> **Why "compile time" for non-virtual overriding:** without `virtual`, the compiler binds the call to a function based solely on the *static type* of the pointer/reference (i.e., what type the pointer is declared as). If `ptr` is `Animal*`, `ptr->speak()` always calls `Animal::speak` — even if the actual object at runtime is a `Dog`. The dispatch decision is baked in at compile time.

---

## Name/Function Hiding (Distinct from Overriding)

In C++, scope lookup happens strictly **by name first**, before the compiler checks parameters, signatures, or function bodies. If a derived class defines _any_ function with a matching name, **all** overloads of that name in the base class become hidden inside the derived class scope.

```cpp
class Base {
public:
    void show(int x)    { cout << "Base::show(int)\n"; }
    void show(double x) { cout << "Base::show(double)\n"; }
};
class Derived : public Base {
public:
    void show(string s) { cout << "Derived::show(string)\n"; }
    // Unhides all overloads of 'show' from the Base class using
    Base::show;
    // Reused the name 'show'. This HIDES both Base::show(int) and Base::show(double).
};
int main() {
    Derived d;
    d.show("hello");   // ✅ Derived::show(string)
    d.show(42);        // ❌ COMPILE ERROR — Base overloads are hidden from direct scope!
    d.show(3.14);      // ❌ COMPILE ERROR 
    d.Base::show(42);  // ✅ Works perfectly manual bypass hidden scope
}
```
#### `using` keyword bypass
```cpp
class Derived : public Base {
public:
    // Unhides all overloads of 'show' from the Base class using
    using Base::show;
    void show(string s) { cout << "Derived::show(string)\n"; }
};
```

#### Base Pointer Bypass
Accessing a `Derived` object through a `Base*` pointer bypasses name hiding. The compiler resolves the call using the **pointer's type**, looking directly inside the `Base` class.
```cpp
Derived d;
Base* ptr = &d; 
d.show();    // ❌ ERROR: Hidden in Derived scope
ptr->show(); // ✅ WORKS: Calls Base::show() directly
```

- Same Name, Same Signature
	  - `virtual` in Base
	    - *Overriding* + Name Hiding
	    - `d.func()` → `Derived::func()`
	    - `d.Base::func()` → `Base::func()`
	    - `Base* ptr = &d; ptr->func()` → `Derived::func()` (Run-Time Dispatch)
	  - No `virtual`
	    -  Name *Hiding*
	    - `d.func()` → `Derived::func()`
	    - `d.Base::func()` → `Base::func()`
	    - `Base* ptr = &d; ptr->func()` → `Base::func()` (Compile-Time Dispatch)
- Same Name, Different Signature (Inheritance)
	  - Name *Hiding*
	  - Derived function hides **all** base overloads.
	  - `d.func(...)` → Only derived signatures are visible.
	  - Access base overloads via `d.Base::func(...)` or `using Base::func;`.
	  - `Base* ptr = &d; ptr->func(...)` → Only base signatures are visible.
	  - No overriding occurs, even if the base function is `virtual`.
- Same Name, Different Signature (Same Class)
	  - Function *Overloading*
	  - All overloads participate in lookup.
	  - Compiler selects the best match.
	  - Always Compile-Time Dispatch.
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
The answer is yes — but only if the *return type is a derived class of the base function's return type*. This is called a **covariant return type**.

Covariant return types only work with *pointers or references* (`Animal*` to `Dog*`) because they track memory addresses, whereas returning **by value** (`Animal` to `Dog`) is forbidden.
```cpp
class Animal {
public:
    virtual Animal* clone() {      // base returns Animal*
        return new Animal();
    }
};
class Dog : public Animal {
public:
    Dog* clone() override { // override returns Dog* — a MORE SPECIFIC type
        return new Dog(); // Dog* is covariant with Animal* (Dog IS-A Animal)
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

