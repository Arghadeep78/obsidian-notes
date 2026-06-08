 ## 1. What is the ER Model?

- The **ER (Entity-Relationship) Model is a data model**. Data models work at the **conceptual/logical layer** — the same layer where the DBA defines data: *what* the data is, *what relationships* exist among data, and *consistency constraints*.
- **Definition:** *"A high-level model based on perception of the real world that consists of a collection of basic objects called **entities** and **relationships** among those objects."*

> The ER Model is used to **visualize** the logical design of the data.

```
  Real World  ──[ER Modelling]──►  ER Diagram (blueprint)  ──[conversion]──►  Tables (Relational Model)

  e.g., a university         Student──enrolls──Course        Student table
        with students        Professor──teaches──Course       Course table
        and courses          ...                              Enroll table
```

---

## 2. Entity & Entity Set

**Entity:**
- A real-world object — e.g., Student, Professor, Vehicle, Customer. (Similar to objects in Object-Oriented technology.)
- An entity is *a thing distinguishable from other things*. (A Student in a college is an entity; the College itself can be an entity.)
- An entity has a **physical existence**.
- An entity has a **unique identification** (a unique attribute = its **Primary Key**). Example: every Student gets a unique Student ID; a Customer has a Customer ID.

**Entity Set:**
- **A set of entities of the same type that share the same properties/attributes.**
- Example: one particular student = an **entity**; the whole `Student` schema/collection = an **entity set**.
- Examples of entity sets: `Student`, `Customer of a Bank`.

```
  Entity Set: Student
  ┌──────────────────────────────────────────┐
  │  { Student1, Student2, Student3, ... }   │
  │                                          │
  │  Student1 = one entity (e.g., "Ram")     │
  │  Student2 = one entity (e.g., "Priya")   │
  └──────────────────────────────────────────┘

  Entity Set ≈ Class, Entity ≈ Object (in OOP)
```

> **Usage note:** While drawing ER diagrams, **"entity" and "entity set" are used interchangeably** — treat them the same when building the model.

---

## 3. Attributes

- **Attributes describe an entity (describe the data).** Without attributes you can't describe an entity.
- The ER Model defines the **"what"** (entity name, e.g., Student) and attributes **describe** that data.

**Examples:**
- `Student`: Student ID, Name, Standard, Course, Batch, Phone Number, Address, …
- `Customer`: Customer ID, Contact Number, Address details, DOB, Email, …

> **Design principle:** Keep attributes **limited and needed** — only those required to describe the entity (make the DB precise, not bulky).

### Domain & Consistency Constraints
- Every attribute has a **domain** = the set of **permitted values** it can take.
- Examples: `Customer Name` cannot be numeric; `Loan` may have a domain {Car Loan, Home Loan, Education Loan} → a "Personal Loan" entry is not allowed.
- **Why constraints help:** they keep the database **consistent**. A badly designed DB without constraints could store a numeric name; on retrieval you'd get an illegal/inconsistent entry.

```
  Attribute: Loan Type
  Domain = { Car Loan, Home Loan, Education Loan }

  Allowed:  "Car Loan"        ✓
  Rejected: "Personal Loan"   ✗  (outside domain → constraint violation)
```

---

## 4. Relationships

- A database stores many entities (e.g., a bank DB has Customers, Loan accounts, Accounts; a university has Professor, Courses, Students). Entities are disjoint, each uniquely identifiable — but there are **relationships** among them.
- A relationship is expressed as a **simple English statement** between entities.

**Examples:**
- `Customer` **borrows** `Loan`
- `Customer` **deposits** / **withdraws** `Account`
- `Customer` **places** `Order`
- `Professor` **teaches** `Course`
- `Student` **enrolls** `Course`
- `Citizen` **has** `Vehicle`
- `Parent` **has** `Child`

```
  Entity          Relationship       Entity
  ──────────      ────────────       ──────────
  Customer   ──── borrows     ────── Loan
  Professor  ──── teaches     ────── Course
  Student    ──── enrolls     ────── Course
  Citizen    ──── has         ────── Vehicle
  Parent     ──── has         ────── Child
```

---

## 5. ER Diagram — Notation

- The ER Model is **graphically represented** as an **ER Diagram**.

| Element | Symbol |
|---------|--------|
| **Entity** | Rectangle box |
| **Relationship** | Diamond |
| **Association** | Line |
| **Attribute** | Ellipse |

**Example:** `Customer` (rectangle) — `Borrow` (diamond) — `Loan` (rectangle), connected with lines.

```
  ER Diagram notation:

  ┌──────────┐         ◇             ┌──────────┐
  │ Customer │ ────── Borrow ─────── │   Loan   │
  └──────────┘                       └──────────┘
       │                                   │
    (ellipse)                          (ellipse)
   Customer ID                         Loan ID
   Name                                Amount
   ...                                 ...

  Legend:
  ┌────────┐  = Entity (rectangle)
  ◇         = Relationship (diamond)
  ───        = Association (line)
  (oval)     = Attribute (ellipse)
```

> **ER Diagram acts as a *BLUEPRINT* for the DB.** Steps: (1) find **entities**, (2) find their **attributes**, (3) establish **relationships** → draw the full ER diagram. Later this blueprint is **converted into tables / the Relational Model**.

---

## 6. Types of Attributes

### 6.1 Simple Attribute
- Cannot be further divided. Example: a customer's bank **account number** (a 10-digit number — taking the first/last 5 digits gives no separate meaning), a student's **roll number**.

### 6.2 Composite Attribute
- **Can be divided into sub-parts.**
- Example: `Name` → First Name, Middle Name, Last Name. `Address` → Street, City, State, Pin Code.
- **Nesting allowed:** a sub-part can itself be composite, e.g., `Street` → Street Name, Street Number.
- **Why divide:** lets you reference a part independently — e.g., fetch all customers whose **Last Name = Mukherjee**, or all citizens in a particular **PIN/ZIP code**. Sub-parts are referable individually **and** together.
- **Interview tip:** When storing `Address`, always make it **composite** — the more composite, the better managed.

```
  Composite Attribute: Address
                Address
               ╱   │   ╲
          Street  City  State  PinCode
          ╱   ╲
  StreetName  StreetNumber   ← nested composite
```

### 6.3 Single-Valued Attribute
- Can have **only one value**. Example: Student ID (737 stays 737), Loan Number of a loan.

### 6.4 Multi-Valued Attribute
- Can have **multiple values**. Example: Phone Number, Nominee Name, Dependent Name.
- **ER notation:** drawn with **double (concentric) ellipses**.
- You can put **limit constraints** (upper/lower bound) — e.g., at most 2 nominees, or at least 1 nominee.

```
  Single-valued:   (Student_ID)      ← one value only
  Multi-valued:   ((Phone_Number))   ← double ellipse; can store multiple values
                    e.g., 9876543210, 9123456789
```

### 6.5 Derived Attribute
- **Derived/calculated from another attribute** — not stored directly.
- Example: `Age` derived from `Date of Birth` (current date − DOB); `Loan Age` / membership period derived from loan disbursal date.
- **ER notation:** drawn with a **dotted ellipse**.

```
  Stored:   (DOB)           ← stored in DB
  Derived:  ·····Age·····   ← dotted ellipse; calculated = today − DOB; not stored
```

---

## 7. NULL Values

A NULL value of an attribute can mean:
1. **Not Applicable** — the value genuinely doesn't apply. Example: a person with no middle name → middle name is NULL = not applicable (not an inconsistency).
2. **Unknown / Missing** — a value should exist but the entry is missing. Example: `Customer Name` is NULL even though the constraint says name must exist → entry missing (an inconsistency/problem).
3. **Not Known (yet)** — value exists but isn't known yet. Example: a new employee whose `Salary` isn't decided yet → salary NULL = not known yet.

```
  NULL can mean three different things:

  Attribute        NULL means...
  ─────────────    ──────────────────────────────────────
  Middle Name      Not Applicable  (person has no middle name — OK)
  Customer Name    Unknown/Missing (should exist but isn't filled — problem)
  Salary           Not Known Yet   (onboarding; will be filled later — OK for now)
```

> What a NULL means depends on the attribute's **type, usage, and the constraints** you defined.

---

## 8. Strong vs Weak Entities

### Strong Entity
- **Independent existence** in the system; can be uniquely identified by its own **Primary Key**.
- Example: a `Student` in a university (identified by Student ID).

### Weak Entity
- **Cannot be uniquely identified by a primary key of its own**; it is **dependent on a strong entity**. Its existence depends on the strong entity existing.
- **Classic example — Loan & Payment:** A bank has many Loans (each Loan = strong entity, has a Loan ID, Loan Type). Each loan has a **Payment** schedule (EMIs). If the Loan doesn't exist, the Payment schedule has no existence → **Payment is a weak entity.**
- A weak entity has a **partial / discriminator key** (e.g., **Payment Number** — a sequential counter 1, 2, 3, …). "Payment Number 1" only uniquely identifies a payment *within* a particular loan.
- **Notation:** Weak entity → double rectangle. Weak relationship → double diamond. The partial/discriminator key of a weak entity → **dotted underline** (vs the solid underline for a primary key of a strong entity).

```
  Strong Entity:          Weak Entity:
  ┌──────────┐            ╔══════════╗
  │  Loan    │            ║ Payment  ║   ← double rectangle
  │ ───────  │            ║ ········ ║   ← dotted underline = partial key
  │ Loan_ID  │            ║ Pay_Num  ║
  └──────────┘            ╚══════════╝
       │                       │
  identified by           identified by Loan_ID + Pay_Num together
  own Loan_ID             (Pay_Num alone is NOT unique across all loans)

  Loan -----◇Loan Payemnt◇═══  Payment   ← concentric double diamond = weak relationship
```

> A weak entity's existence exists only when the related strong entity (e.g., the Loan) exists.

---

## 9. Strong vs Weak Relationships

- **Strong relationship:** both participating entities have independent existence (own primary keys). Examples: `Customer places Order` (Customer ID + Order ID), `Professor teaches Course` (Professor ID + Course ID).
- **Weak relationship:** connects a weak entity to its strong entity (e.g., Loan ↔ Payment); drawn with a **concentric (double) diamond**.

```
  Strong Relationship:          Weak Relationship:
  ┌──────────┐       ◇◇         ╔═════════╗
  │ Customer │───── Loan  ══════╣ Payment ║
  └──────────┘                  ╚═════════╝
   (own PK)       (own PK)      (no own PK)
```

---

## 10. Degree of a Relationship

**Degree = number of entities participating in a relationship.**

### Unary Relationship (1 participant)
- One entity set related to itself. Example: `Employee` **manages** `Employee` (a manager is also an employee). *(Rare.)*

### Binary Relationship (2 participants) — most common
- Examples: `Student takes Course`, `Customer borrows Loan`, `Customer deposits Account`, `Professor teaches Course`.

### Ternary Relationship (3 participants) — rare
- Example: `Employee` **works-on** relating **Employee + Branch + Job**. An employee works on a branch; the employee has a job role; the job role exists in that branch → all three are related in one relationship.

```
  Unary (degree 1):              Binary (degree 2):        Ternary (degree 3):
                                                              ┌──────────┐
  ┌──────────┐                  ┌─────────┐                  │ Employee │
  │ Employee │◄──┐              │ Student │                  └────┬─────┘
  └────┬─────┘   │              └────┬────┘                       │
       │         │                   │                             │
    ◇manages     │               ◇ takes                     ◇ works-on
       │         │                   │                       ╱         ╲
       └─────────┘              ┌────┘                  ┌──────┐    ┌─────┐
  (Employee manages             │                       │Branch│    │ Job │
   another Employee;           ┌──────┐                └──────┘    └─────┘
   manager IS also an          │Course│
   employee — same entity set) └──────┘

  Note: in Unary, the entity participates in a relationship WITH ITSELF.
  The diamond connects back to the same rectangle (one entity set, two roles).
```

> Most relationships are **binary**; unary and ternary are uncommon.

---

## 11. Relationship Constraints — Mapping Cardinality

**Mapping Cardinality = the number of entities to which another entity can be associated via a relationship.**

### 11.1 One-to-One (1:1)
- One entity in A associates with **at most one** entity in B, and vice versa.
- Example: `Citizen has Aadhaar Card` (one citizen → one Aadhaar).

```
  1:1  Citizen ────────────── Aadhaar
       C1  ─────────────────  A1
       C2  ─────────────────  A2
       C3  ─────────────────  A3
```

### 11.2 One-to-Many (1:N)
- One entity in A associates with **many** entities in B; but each entity in B associates with **at most one** entity in A.
- Example: `Citizen has Car` (one citizen can own multiple cars; each car is owned by exactly one citizen).

```
  1:N  Citizen ────────────── Car
       C1  ──┬──────────────  Car1
             ├──────────────  Car2
             └──────────────  Car3
       C2  ────────────────── Car4
```

### 11.3 Many-to-One (N:1)
- Many entities in A associate with **one** entity in B; one entity in B associates with many in A.
- This is the mirror of 1:N — depends on which side you label A and which you label B.
- Example: `Course taught by Professor` — many courses map to one professor; one professor can teach many courses.

```
  N:1  Course ────────────── Professor
       Course1  ──┐
       Course2  ──┼──────── Professor1
       Course3  ──┘
       Course4  ─────────── Professor2
```

### 11.4 Many-to-Many (M:N)
- One entity in A associates with many in B, **and** one entity in B associates with many in A.
- Examples: `Customer buys Product`; `Student attends Courses`.

```
  M:N  Student ────────────── Course
       S1  ──┬───────────────  C1
             └───────────────  C2
       S2  ──┬───────────────  C1
             └───────────────  C3
       S3  ────────────────── C2
  (one student, many courses; one course, many students)
```

---

## 12. Participation Constraints (aka Minimum Cardinality Constraint)

#### Two types:

### Total Participation
- **Every** entity in the entity set participates in **at least one** relationship instance.
- **Notation:** **double lines** from the entity to the relationship.
- Example: In `Customer borrows Loan`, the **Loan side has total participation** — every loan must be borrowed by some customer. No orphan loan that no customer took.

### Partial Participation
- **Not all** entities in the set need participate in the relationship.
- Example: In `Customer borrows Loan`, the **Customer side is partial** — there can be customers who haven't taken any loan.

```
  Participation in "Customer borrows Loan":
  
  Customer ────────────────── ◇ borrows ══════════════ Loan
  (single line = partial)              (double line = total)

  Partial: some customers have no loan (OK)
  Total:   every loan must have at least one customer (mandatory)
```

> The **Customer side is partial** (a customer need not have a loan), but the **Loan side is total** (every loan must be borrowed by at least one customer).

### Key rule
- **Weak entities ALWAYS come with a Total Participation constraint** (a weak entity, e.g., Payment, exists only when tied to its strong entity, e.g., Loan).
- **Strong entities may have total OR partial participation.** In `Customer borrows Loan`, both are strong entities; Customer's participation is partial, Loan's is total.

---

## 13. Summary of ER Notations

![[Pasted image 20260606150623.png]]
Rect, ellipse, diamond