# Part 1 — Constructors, `this` Pointer & Copy Semantics

## Constructor 

### What Is a Constructor?

- A *special method* invokes called the **constructor** runs *automatically* on that memory

When you write `Teacher t1;`, two things happen in one shot:
1. **Memory** is <u>allocated</u> for `t1` (see [[01 - Introduction, OOP & Pointers#Stack vs Heap — Two Ways to Create Objects|Stack vs Heap]] for memory details)
2. **constructor** runs *automatically*

A constructor is a special member function that:
- Has the **same name** as the class
- Has **no return type** (not even `void`)
- Runs **automatically** only once (per object) the moment an object is created. (used to **initialise** the object's data members)
- Memory allocation occust when constructor is called.

If you don't write any constructor, the compiler silently generates a do-nothing one for you. The moment you write *any* constructor yourself, the compiler stops generating that default.

#### 3 ways to write a constructor

1. *Member Initializer List* (Preferred & Modern)
```cpp 
Student(int a, double cgpa) : age(a), cgpaPtr(new double(cgpa)) {}
```

2. *Normal Constructor Body*
```cpp 
Student(int a, double cgpa) {  
	age = a;  
	cgpaPtr = new double(cgpa);  
}
```

Initializer lists are preferred because members are **initialized directly** instead of being created first and then assigned.
- `age` is initialized first (with an unspecified/default/garbage value).
- `cgpaPtr` is initialized first (with an unspecified value).
- Then assignments happen inside the constructor body.

3. *Aggregate Initialization*

An **aggregate** is a class/struct with no
- user-declared constructors (including default, delete(c++ 20 and beyond))
- no private/protected non-static data members
- no base classes
- no virtual functions *(if these 4 violated can't use) (destructors are allowed)*
Aggregates support **aggregate initialization**: initializing directly from a brace list without needing a constructor:

```cpp
class Student { or //Stuct(public only)
  public:
    int age;
    string name;
};

Student s{20, "Arghadeep"};
```

---

### Non-Parameterized (Default) Constructor

Use this when all objects should start with the same default values:

```cpp
class Teacher {
public:
    string name, department, subject;
    double salary;

    // Constructor — same name as class, no return type
    Teacher() {
        // This runs automatically when any Teacher object is created
        department = "Computer Science";   // set a default value
        salary = 0;
        cout << "Constructor called!\n";
    }
};

int main() {
    Teacher t1;   // prints: Constructor called! — automatic, you didn't call it
    Teacher t2;   // prints: Constructor called! — one call per object
    cout << t1.department << "\n";   // Computer Science — set by constructor
    cout << t2.department << "\n";   // Computer Science — t2 also got it
}
```

---

### Parameterized Constructor

Use this when each object needs different initial values. Pass them at creation time:

```cpp
class Teacher {
public:
    string name, department, subject;
    double salary;
    // Parameterized constructor — takes arguments at object creation
    Teacher(string name, string department, string subject, double salary=1000) {  //here 100 set as default used if salary not passed as argument
        // PROBLEM: parameter names clash with member names!
        // "name = name;" would be assigning the parameter to itself — a bug!
        // FIX: use "this->" to explicitly refer to the OBJECT's member
        this->name       = name;          // object's name = parameter name
        this->department = department;
        this->subject    = subject;
        this->salary     = salary;
    }
    void getInfo() {
        cout << name << " | " << department << " | " << subject
             << " | " << salary << "\n";
    }
};
int main() {
    // Values passed right at creation — object is fully set up instantly
    Teacher t1("Shraddha", "Computer Science", "C++", 25000);
    t1.getInfo();   // Shraddha | Computer Science | C++ | 25000
}
```

**Common trap:** If you define only a parameterized constructor, then `Teacher t1;` (no arguments) will fail to compile — the compiler no longer generates a default constructor. *Either add a default constructor explicitly, or use default parameter values.*

Always declare as ==public==.

>**Note**: **Parameters** are *variables* in a **function** **definition** wj **arguments** are the actual values **passed** to those parameters during a function call.

---

### Constructor Overloading

You can have multiple constructors in the same class — the compiler picks the right one based on the arguments:

- example of [[05 - Polymorphism — Overloading, Overriding & Virtual Functions | Polymorphism]]

```cpp
class Teacher {
public:
    string name, department;

    Teacher() {                        // no-arg version
        department = "Computer Science";
        cout << "Default constructor\n";
    }

    Teacher(string n, string d) {      // full version
        name = n;
        department = d;
        cout << "Parameterized constructor\n";
    }

    Teacher(const Teacher& obj) {      // copy version (covered later)
        name = obj.name;
        department = obj.department;
        cout << "Copy constructor\n";
    }
};

int main() {
    Teacher t1;                   // calls the no-arg version
    Teacher t2("Shraddha", "CS"); // calls the full version
    Teacher t3(t2);               // calls the copy version
}
```

**Classic interview trap:** `Teacher t1();` looks like it creates an object, but it actually **declares a function** named `t1` that returns a `Teacher`. This is called the **Most Vexing Parse**. Always write `Teacher t1;` (no parentheses) for the default constructor.

---

## `this` Pointer

Every time you call a member function, C++ passes a hidden pointer, `this` to that function *pointing* to the **object it was called on**.

When you write `t1.setName("Shraddha")`, C++ internally transforms it to `Teacher::setName(&t1, "Shraddha")`. Inside `setName`, `this == &t1`.

- `this` is of type `ClassName* const` — the pointer itself can't be changed
- `this->member` is how you access the object's own data from inside a function (in class definition)
- `*this` is the entire object itself (useful for returning self)

**`this` does NOT exist in [[07 - The static Keyword in C++ OOPs#Part 1 — Static Variables, Members & Functions|Static]] functions** — static functions belong to the class, not any object.

```cpp
#include <iostream>
#include <string>
using namespace std;

class Teacher {
public:
    string name, department;
    // Use case 1: Disambiguate when parameter name matches member name
    Teacher(string name, string department) {
        this->name = name;        // "this->name" = object's member
        this->department = department;  // "name" alone = the parameter
    }

    // Use case 2: Method chaining — return the object itself so calls can be chained
    Teacher& setName(string name) {
        this->name = name;
        return *this;   // return the whole object by reference
    }
    Teacher& setDept(string dept) {
        this->department = dept;
        return *this;
    }
    void print() { cout << name << " | " << department << "\n"; }
};
int main() {
    Teacher t1("Shraddha", "CS");
    t1.print();
    // Method chaining: each setter returns *this, so you can chain calls
    Teacher t2("", "");
    t2.setName("Rahul").setDept("EE").print();
    //         ↑ returns t2     ↑ called on t2    ↑ called on t2
}
```

---

## Copy Constructor

### What Is a Copy Constructor?

A copy constructor creates a **new object as a copy of an existing one**. Its signature is always:
```cpp
Teacher(const Teacher& obj)
//passed by reference
```

It fires in three situations:
1. `Teacher t2(t1);` — explicit copy construction *default copy constructor of c++*
2. `Teacher t2 = t1;` — copy-initialization (looks like assignment but calls copy constructor *(same as above)*
3. *Passing* or returning an *object by value* in a function

- **Why `const`?** — The copy constructor should only **read** from the source object and copy its data (not compulsory).

- **Why `&` (reference)?** — CRITICAL. If you took by value instead of reference, the compiler would need to copy the argument to pass it... which would call the copy constructor... which would copy its argument... infinite recursion → *==stack overflow*==. Reference avoids the copy.

```cpp
class Teacher {
public:
    string name, department, subject;
    double salary;

    Teacher(string name, string department, string subject, double salary) {
        this->name = name; this->department = department;
        this->subject = subject; this->salary = salary;
    } //constructor

    // Custom copy constructor
    Teacher(const Teacher& original) {
        // Copy each member from the original object
        this->name       = original.name;
        this->department = original.department;
        this->subject    = original.subject;
        this->salary     = original.salary;
        cout << "Copy constructor called!\n";
    } // Custom copy constructor
```

 **Memory** in Copy Constructors:
 
> ┌──────────────────┐  ← Teacher t1 (starts here)
│  name  (string)  │
│  dept  (string)  │
│  salary(double)  │
└──────────────────┘
>
>┌──────────────────┐  ← Teacher t2 (entirely separate block)
│  name  (string)  │
│  dept  (string)  │
│  salary(double)  │
└──────────────────┘
>
>Memory is *duplicated* to a new block

---

## Shallow Copy vs Deep Copy

### Shallow Copy

The **default copy constructor** does a **shallow (member-wise) copy** — it copies each member's value literally. (using this->name = obj->name)

For simple types (`int`, `double`, `string`) this is fine. For **pointer members**, this is dangerous:

```
Object s1:  name = "Rahul"    cgpaPtr → [address 5000] → value 8.9
                                                ↑
Shallow copy just copies the address (5000), not the value
                                                ↓
Object s2:  name = "Rahul"    cgpaPtr → [address 5000] → value 8.9
                                          SAME address!
```

Both `s1` and `s2` point to the **exact same memory location**. This causes two problems:
1. Modifying `*s2.cgpaPtr` changes `s1`'s data too — they're *shared*
2. When both are destroyed, both destructors try to `delete` the same address → **double free → crash** 

> - The pointer does not know it was freed.
> - The memory allocator knows whether a memory block is allocated or already  freed. causing double free to lead to undefined behaviour.
>  - deleting nullptr is safe in c++.

*Note*: ==new== always creates ==pointer==


```cpp
class Student {
public:
    string name;
    double* cgpaPtr;
    Student(string name, double cgpa) {
        this->name = name;
        cgpaPtr = new double(cgpa);
    }
    // Shallow Copy: copies pointer address, not data
    Student(const Student& obj) {
        name = obj.name;
        cgpaPtr = obj.cgpaPtr;   // both objects share same heap block (memory) *new*
    }
    ~Student() {
        delete cgpaPtr;
 // Destruction: At scope exit automatically called
// both objects try to delete same memory → double free
//unfined behaviour
    }
};
int main() {
    Student s1("Rahul", 8.9);
    Student s2(s1);      // s1.cgpaPtr == s2.cgpaPtr
    *s2.cgpaPtr = 9.2;   // changes s1's CGPA too
}
```

---

### Deep Copy — The Correct Solution

>Use when dynamically allocated memory involved (**heap memory**)

In a deep copy, for every pointer member, **allocate brand-new memory** and copy the *value* (not the address) into it. Each object owns its own independent heap block.

```cpp
class Student {
public:
    string name;
    double* cgpaPtr;
    // DEEP copy constructor — correct way
    Student(const Student& obj) {
        name    = obj.name;
        cgpaPtr = new double;         // NEW allocation — brand new memory block
        *cgpaPtr = *obj.cgpaPtr;      // copy the VALUE at that address, not the address itself
        cout << "Deep copy constructor called!\n";
    }
    void getInfo() {
        cout << name << " | CGPA: " << *cgpaPtr << "\n";
    }
    ~Student() {
        delete cgpaPtr;   // each object frees its own block — no conflict
    }
};

int main() {
    Student s1("Rahul Kumar", 8.9);
    Student s2(s1);          // deep copy — s2 gets its own separate heap block
    s2.name    = "Neha Kumar";
    *s2.cgpaPtr = 9.2;       // only s2's memory changes — s1 is untouched
    s1.getInfo();   // Rahul Kumar | CGPA: 8.9  ← unchanged ✅
    s2.getInfo();   // Neha Kumar  | CGPA: 9.2
}
```

**Memory layout after deep copy:**
```
s1:  name="Rahul"   cgpaPtr → [address 5000] → 8.9   (s1's own block)
s2:  name="Neha"    cgpaPtr → [address 6000] → 9.2   (s2's own NEW block)
```
Completely independent. No sharing.

## Move Constructor

A **move constructor** transfers ownership of resources (heap memory, file handles) from a temporary source object to a new object, avoiding expensive deep copies. It "steals" pointers and *nulls out the source* (sets source to valid empty state (no double-free by destructor).
It used *&&* (rvalue reference)
By *default*: we get a *shallow* move constructor & assignment member function.

 *`noexcept`* : tells the STL (e.g., `std::vector`) that the move won't throw. Without this, `std::vector` plays it safe and copies instead of moves during reallocation.

```cpp
class Car {  
	int* ptr;  
public:  
	Car(int x) {  
		ptr = new int(x);  
	}  
	Car(Car&& other) noexcept {  //usually noexcept is given
		ptr = other.ptr; // take ownership  
		other.ptr = nullptr; // leave source safe  
	}   
	//or A(A&&) = default;
	~Car() {  
		delete ptr;  
	}  
	
	Car& operator=(Car&& other) noexcept {  //move assignment
		if (this != &other) {  
			delete ptr; // free current resource  
			ptr = other.ptr; // take ownership  
			other.ptr = nullptr;  
		}  
		return *this;  
	}
	
};
int main() {  
	Car c1(10), c2(20);  
	Car c3(std::move(c1)); // move constructor called  
	car c3 = std::move(c2); //move assignement
	return 0;  
}
```

check [[08 - Move Semantics & Rule of Five (above C++11)#Part 1 — lvalue/rvalue, Move Constructor & Move Assignment||for more details]]

### Pointer Allocation Behavior

- **`Node* node = new Node[5]`**: Allocates a contiguous block for 5 `Node` objects, sequentially calls the default constructor for each, and requires `delete[] node;`. (only *default constructor*)
- **`Node* node = new Node()`**: Allocates a single `Node` object, triggers value-initialization (zero-initializing scalar members), and requires `delete node;`.
	In both case \*node points to the first/only object's memory address
- **`Node node[2] = { Node(10), Node(30)}`** here also *node* decays to &node\[0\].

---

# Part 2 — Destructor, Rule of Three & Special Topics

## Destructor

### What Is a Destructor?

A destructor is the exact counterpart to a constructor — it runs automatically when an object's **lifetime ends** (goes out of scope, or `delete` is called on a heap object).

1. `~` tilde prefix
2. no return type
3. no parameters
4. can't be overloaded

> automatically called on **reaching out of scope**

```cpp
	~ClassName(){
	}   // tilde prefix, no return type, no parameters, cannot be overloaded
```

The **default** compiler-generated destructor only cleans up stack-allocated members. It does **not** `delete` any pointer members — those *heap blocks are YOUR responsibility.*

- Manual destructor used in very low level development
```cpp
student1.~Student(); // Manual destructor call
```

```cpp
class Student {
public:
    string name;
    double* cgpaPtr;   // heap pointer — we must clean this up ourselves
    ~Student() {
        delete cgpaPtr;   // free the heap memory — MANDATORY if we allocated it
        cout << "Destructor: " << name << " destroyed\n";
    }
};
int main() {
    {//new scope
        Student s1("Rahul Kumar", 8.9);
        s1.getInfo();
    }   // ← destructor called HERE automatically, the moment s1 goes out of scope
        //   NOT at the end of main() — at the end of the BLOCK
    cout << "After the block ends\n";
    return 0;
}
// Output:
// Constructor: Rahul Kumar created
// Rahul Kumar | CGPA: 8.9
// Destructor: Rahul Kumar destroyed   ← fires at end of { } block
// After the block ends
```

> for memory link check out [[01 - Introduction, OOP & Pointers#Stack vs Heap — Two Ways to Create Objects | here]]

### Virtual Destructor

A virtual destructor ensures that when a *derived object* is *deleted* *through* a *base-class pointer*, *both* the derived and base *destructors* are *called*.

If the base destructor is NOT virtual, only the base destructor runs — the derived destructor is skipped → *resource leak*.

```cpp title:code
class BaseTeacher {
public:
    virtual ~BaseTeacher() {
        cout << "Base Destructor\n";
    }
};

class DerivedTeacher : public BaseTeacher {
public:
    ~DerivedTeacher() {
        cout << "Derived Destructor\n";
    }
};

Base* ptr = new DerivedTeacher(); //base class pointer pointing to a derived class object.
delete ptr;
```

``` text title=output
Derived Destructor
Base Destructor
```

Why `virtual`?

- Ensures the **derived destructor runs first**.
- Prevents resource leaks when deleting derived objects through base pointers.
### Without Virtual Destructor

``` text title=output
Base Destructor
```

Only `Base` destructor may run, causing resource leaks in `Derived`.
### Rule
```text
If a class is intended to be used as a base class, make its base class destructor virtual.
```

---

## Rule of Three — The Golden Rule

> **If your class defines any one of these three, you almost certainly need all three:**

| Special Member           | When It Runs                                                            |
| ------------------------ | ----------------------------------------------------------------------- |
| Destructor               | Object goes out of scope or `delete` is called                          |
| Copy Constructor         | `Student s2(s1)`, `Student s2 = s1`, pass/return by value               |
| Copy Assignment Operator | **Existing object is assigned** the value of another object (`s2 = s1`) |

**Why all three?** A class with a heap pointer needs:
- Destructor → to free the heap memory
- Copy Constructor → to prevent shallow copy on initialization
- Copy Assignment → to prevent shallow copy on reassignment

Missing any one of the three leads to either **double-free** or **memory leak**.

```cpp
// Copy Assignment Operator (the third member of the Rule of Three)
Student& operator=(const Student& other) {
    if (this == &other) return *this;   // s1 = s1 (self-assignment) guard
    // operator= is a specail function name (returns *this i.e. Student&)
    delete cgpaPtr;                     // free what we currently own
    cgpaPtr = new double;                   // deep copy — allocate new memory
    *cgpaPtr = *other.cgpaPtr;             // copy the VALUE (same two-step as copy constructor)
    name    = other.name;
    return *this;   // return *this to allow chaining: s1 = s2 = s3;
}
```

>Note: **\*this** is basically **class&** i.e. dereferencing a pointer to class gives object reference

Student& as we are returning reference to original to itself:
allows chaining `s1 = s2 = s3;` equivalent to `s1 = (s2 = s3);`, `(s2 = s3)` return `s2` itself by reference

---

## Rapid-Fire Interview Q&A

| Question                           | Answer                                                                                                                                                                           |     |
| ---------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --- |
| What is a constructor?             | Special member function, same name as class, no return type, auto-called at object creation to initialise members.                                                               |     |
| Can a constructor be `virtual`?    | No — vtable isn't set up until construction completes, so `virtual` constructor makes no sense.                                                                                  |     |
| `Teacher t1()` vs `Teacher t1;`?   | `t1()` is a *function declaration* (Most Vexing Parse). `t1;` creates an object.                                                                                                 |     |
| Why `const&` in copy constructor?  | (1) Reference avoids infinite recursion from passing by value. (2) `const` prevents accidental mutation of the original.                                                         |     |
| Shallow vs deep copy?              | Shallow copies the pointer's address (both objects share heap → double-free risk). Deep allocates new memory and copies the value (independent → safe).                          |     |
| Rule of Three?                     | If you define any of destructor, copy constructor, or copy assignment — define all three. Class with heap pointer needs all three.                                               |     |
| Why `virtual` destructor in base?  | Without it, `delete base_ptr` on a derived object skips the derived destructor → resource leak + undefined behaviour.                                                            |     |
| What is *RAII*?                    | *Resource Acquisition Is Initialisation* — acquire resources in constructor, release in destructor. This ensures cleanup happens automatically when an object goes out of scope. |     |
| After `delete ptr`, what is `ptr`? | A dangling pointer — pointing to freed memory. Always set `ptr = nullptr` after deleting.                                                                                        |     |
| `delete` vs `delete[]`?            | `delete` for a single object (`new T`). `delete[]` for arrays (`new T[n]`). Wrong pairing = undefined behaviour.                                                                 |     |
