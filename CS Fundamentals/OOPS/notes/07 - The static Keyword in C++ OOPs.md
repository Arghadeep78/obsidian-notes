# Part 1 — Static Variables, Members & Functions

## What Does `static` Do? (The Core Idea)

Normally in C++, a variable's lifetime is tied to its scope — a local variable is born when a function starts and dies when it returns. An object's data members exist for as long as the object lives.

`static` breaks this rule. It says: **"this thing lives longer than normal — it belongs to the program's lifetime (or the class's lifetime), not to any individual function call or object."**

**Three uses of `static` in OOPs:**
1. Inside a function — variable persists across calls
2. Inside a class — variable/function belongs to the class, not any individual object
3. On an object — object survives beyond its enclosing scope

---

## 1. Static Local Variable — Remembers Between Calls

Without `static`, a local variable is re-created and re-initialized every time the function is called:

```cpp
#include <iostream>
using namespace std;

void counter() {
    int x = 0;     // created fresh every call, initialized to 0 every time
    cout << x << " ";
    x++;           // incremented, but thrown away when function returns
}

int main() {
    counter();   // prints: 0
    counter();   // prints: 0  ← x was re-created from scratch
    counter();   // prints: 0
}
```

With `static`, the variable is created **once**, and its value persists between calls:

```cpp
void counter() {
    static int x = 0;   // created ONCE (first call), never re-initialized again
    cout << x << " ";
    x++;                // increment persists — survives function return
}

int main() {
    counter();   // prints: 0
    counter();   // prints: 1  ← x remembered it was 1 from last call
    counter();   // prints: 2
}
```

**Key rules:**
- `static int x = 0;` — the `= 0` initialization runs exactly ONCE (on first call)
- The variable lives in **static storage** (not the stack) — survives function returns
- Scope is still local (only visible inside the function — but its lifetime is the whole program)

---

## Where Do Static Variables Live in Memory?

See [[01 - Introduction, OOP & Pointers#Memory Layout — The Full Picture|memory layout]] for a complete picture.

```
Program memory layout:
┌──────────────────────┐
│     Stack            │  ← local variables, function call frames
│  (grows/shrinks)     │
├──────────────────────┤
│     Heap             │  ← dynamic allocations (new/delete)
│  (manual management) │
├──────────────────────┤
│  Static / Global     │  ← static variables live HERE
│  Storage             │    allocated at program start
│  (fixed size)        │    freed only when program ends
└──────────────────────┘
```

---

## 2. Static Data Member — One Variable Shared by All Objects

Normally, each object has its own independent copy of every data member:

```cpp
class Teacher {
    public:
        string name;   // each Teacher object has its own name
};

Teacher t1, t2;
t1.name = "Alice";
t2.name = "Bob";    // t1.name is still "Alice" — completely independent
```

But sometimes you want a variable that is **shared by ALL objects** — like a counter tracking how many objects exist. For this, use `static`:

```cpp
class Student {
    public:
        string name;
        static int count;   // ONE copy shared by ALL Student objects
                            // not in any single object's memory — belongs to the class

        Student(string n) : name(n) {
            count++;    // every new Student increments the SHARED counter
        }

        ~Student() {
            count--;    // every destroyed Student decrements it
        }
};

// MANDATORY: define (and optionally initialize) the static member OUTSIDE the class
// This is where the actual memory is allocated for it.
// WHY outside? The class definition lives in a header included by many .cpp files.
// If the static member were defined inside the class body, every translation unit
// that includes the header would produce its own definition → ODR (One Definition Rule) violation → linker error.
// Defining it once in a .cpp file guarantees exactly one memory location across the whole program.
int Student::count = 0;

int main() {
    Student s1("Alice");   // count = 1
    Student s2("Bob");     // count = 2
    {
        Student s3("Charlie");   // count = 3
    }                            // s3 destroyed → count = 2

    cout << Student::count << "\n";   // 2 — access via class name (preferred)
    cout << s1.count << "\n";         // 2 — also valid, but class name is clearer
}
```

**Key rules for static data members:**
- **Declared** inside class with `static` keyword
- **Defined** (memory allocated) OUTSIDE class: `int Student::count = 0;`
- **Accessed** via `ClassName::member` (preferred) or through any object
- All objects **share the same** memory location — one variable for the whole class
- Exists even if **zero objects** are created

---

## 3. Static Member Functions — Callable Without Any Object

A static member function belongs to the **class**, not to any individual object. You can call it without creating any object at all.

```cpp
class MathUtils {
    public:
        static int add(int a, int b) {   // static — no object needed
            return a + b;
        }

        static double pi() {
            return 3.14159265;
        }
};

int main() {
    // Call directly on the class — no object created!
    cout << MathUtils::add(3, 4);   // 7
    cout << MathUtils::pi();        // 3.14159265
}
```

**Critical restriction:** Static functions have **no `this` pointer** (they're not called on any object). So they can ONLY access other static members:

```cpp
class A {
    int x;             // non-static — belongs to individual objects
    static int y;      // static — belongs to the class

    static void foo() {
        // cout << x;  ← ❌ ERROR: 'x' is per-object, but static fn has no object (no 'this')
        cout << y;     // ✅ OK — y is static, accessible without an object
    }
};
```

---

## Object Counter — The Classic Use Case

Combining static member + static function to count live objects:

```cpp
#include <iostream>
using namespace std;

class Student {
    private:
        string name;
        static int count;   // shared counter — how many Student objects currently exist

    public:
        Student(string n) : name(n) {
            count++;
            cout << "Created: " << name << " | Total students: " << count << "\n";
        }

        ~Student() {
            count--;
            cout << "Destroyed: " << name << " | Total students: " << count << "\n";
        }

        // Static function to query the static member — no object needed to call this
        static int getCount() {
            return count;
        }
};

int Student::count = 0;   // definition outside class

int main() {
    Student s1("Alice");    // Created: Alice | Total: 1
    Student s2("Bob");      // Created: Bob   | Total: 2
    {
        Student s3("Charlie");   // Created: Charlie | Total: 3
    }                            // Destroyed: Charlie | Total: 2

    cout << "Current count: " << Student::getCount() << "\n";   // 2
}
```

---

## 4. Static Object — Extended Lifetime

A `static` object inside a block survives beyond the block's scope:

```cpp
// Without static — object destroyed when block exits
if (true) {
    MyClass obj;   // normal object
}   // obj is destroyed HERE — at end of the if block

// With static — object survives the block
if (true) {
    static MyClass obj;   // static object — created once, NOT destroyed at block exit
}   // obj is NOT destroyed here — persists

// obj is destroyed only when the PROGRAM ENDS (after main returns)
```

Practical demonstration:
```cpp
#include <iostream>
using namespace std;

class ABC {
    public:
        ABC()  { cout << "Constructor called\n"; }
        ~ABC() { cout << "Destructor called\n"; }
};

int main() {
    cout << "Before if-block\n";
    if (true) {
        static ABC obj;   // created here (once)
    }                     // NOT destroyed here
    cout << "After if-block\n";
    return 0;
}   // obj destroyed HERE — when main ends

// Output:
// Before if-block
// Constructor called
// After if-block
// Destructor called     ← fires when program ends, not when if-block ends
```

---

## Summary Table

| Context | What It Does |
|---|---|
| `static` local variable | Initialized once, persists between function calls |
| `static` data member | Shared by ALL objects; one memory location per class |
| `static` member function | Belongs to class, not any object; no `this`; callable without object |
| `static` object | Survives its enclosing scope; destroyed only when program ends |

---

# Part 2 — Singleton Pattern & Interview Q&A

## Singleton Pattern — Static's Most Famous Use

The Singleton pattern uses static to ensure only ONE instance of a class ever exists:

```cpp
class Config {
    private:
        Config() {}   // private constructor — nobody can do 'Config c;' directly

    public:
        static Config& getInstance() {
            static Config inst;   // created once on first call, lives forever
            return inst;          // always returns the SAME object
        }

        void set(string key, string val) { /* ... */ }
        string get(string key) { /* ... */ }
};

int main() {
    Config& c1 = Config::getInstance();   // creates the one instance
    Config& c2 = Config::getInstance();   // returns the SAME instance (no new creation)
    // c1 and c2 refer to the exact same object
}
```

**Common interview follow-up — "Is this Singleton thread-safe?"**

Yes, since C++11. The standard guarantees that initialization of a static local variable is performed **exactly once**, even if multiple threads reach the initialization point simultaneously. The compiler inserts the necessary synchronization automatically. This is sometimes called *"magic statics"*.

```cpp
// This is thread-safe in C++11 and later — no manual mutex needed:
static Config inst;   // compiler ensures only one thread initializes this
```

---

## Interview Q&A

**Q: "What is a static variable?"**
A: A variable initialized once and whose value persists for the entire program lifetime. In a function, it survives between calls. In a class, it's shared by all objects.

**Q: "What is a static data member?"**
A: A class-level variable shared by all objects of that class — one memory location for the whole class. Must be defined outside the class. Exists even when no objects have been created.

**Q: "What is a static member function?"**
A: A function that belongs to the class (not any object). Called via `ClassName::function()`. Cannot access non-static members (no `this` pointer).

**Q: "What is the difference between static and non-static member?"**
A: Non-static: each object has its own copy, tied to that object's lifetime. Static: single shared copy for all objects, tied to program lifetime.

**Q: "Can you access a static member without creating an object?"**
A: Yes — via `ClassName::member`. This is the preferred way.

**Q: "Why can't a static function access non-static members?"**
A: Static functions have no `this` pointer — they aren't called on any specific object, so there's no "current object" whose members they could access.

**Q: "When is a static local variable initialized?"**
A: On the first time execution reaches that line. It's initialized exactly once, even if the function is called millions of times.

**Q: "Why must a static data member be defined outside the class?"**
A: The class definition typically lives in a header included by many `.cpp` files. Defining the static member inside the class would create a separate definition in every translation unit that includes the header — an ODR (One Definition Rule) violation that causes a linker error. The out-of-class definition in a single `.cpp` file ensures exactly one memory location across the whole program.

**Q: "Is the Singleton (static local variable pattern) thread-safe?"**
A: Yes, since C++11. The standard guarantees that a static local variable is initialized exactly once, even under concurrent access — the compiler inserts the necessary synchronization automatically. This is sometimes called "magic statics". In C++03 it was NOT thread-safe and required a double-checked locking pattern with a mutex.
