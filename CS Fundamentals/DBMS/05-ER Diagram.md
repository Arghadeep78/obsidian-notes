## 0. Recap & Goal

Last two lectures covered the full ER Model: relationships, entity sets, constraints applied on relationships (**mapping cardinality**, **participation constraints**), and ER notations. This lecture = **Steps to make an ER Diagram** for any system (university, school, banking — any use case).

> The ER Model / ER Diagram is used to build the **Conceptual Schema**.

```
  ER Diagram  ──────────────────────────────────────────────────────────►  Relational Tables
  (Conceptual Schema / logical blueprint)          (implemented in MySQL, PostgreSQL, etc.)

  This lecture: HOW to build the ER Diagram systematically.
```

---

## 1. The 3 Steps to Make an ER Diagram

Follow these 3 steps for **any** system and the ER diagram comes out easily:

### Step 1 — Identify Entity Sets

Think: which entities / entity sets exist in the system.

### Step 2 — Identify Attributes (and their types)

For each entity, decide attributes and their **types** (single-valued, multi-valued, derived, composite) — and the **Primary Key**.

### Step 3 — Identify Relationships & Constraints (MOST IMPORTANT)

- Which relationships to establish (strong / weak).
- **Degree** of the relationship.
- **Mapping Cardinality** (1:1, 1:N, N:1, M:N).
- **Participation Constraint** (Total vs Partial).

```
  3-Step ER Formulation Process:

  Step 1: Entities          Step 2: Attributes        Step 3: Relationships
  ───────────────           ──────────────────        ─────────────────────────
  What "things" exist?      What describes them?      How are they connected?
  Student, Loan, Branch     Name, ID, Amount ...      borrows, teaches, enrolls
                            + types (composite,       + cardinality (1:1, M:N)
                              derived, multi-val)     + participation (total/partial)
                            + Primary Key             ← MOST IMPORTANT STEP
```

> **Interview tip:** Many people draw a simple ER diagram but forget to draw **mapping cardinality** and **participation constraints** — these are the most important from an interview perspective. Always add them.

---

## 2. Before designing: Requirement Specification

Before building the ER model, **collect DB requirements** — what the DB user/stakeholders want (separate customer data, a loan DB, an accounts DB, etc.). This is **Requirement Engineering** (from Software Engineering): interview the DB user and stakeholders to remove ambiguity.

> **In an interview** you don't have time for full requirement engineering, so you **assume (अंश/assumptions)** many requirements yourself and build the diagram with those assumptions.

```
  Real world:   Requirement Engineering  →  clear requirements  →  ER Design
  Interview:    State your assumptions upfront  →  then design directly
                ("I'm assuming a customer can hold multiple accounts ...")
```

---

## 3. Worked Example — Banking System (the example built in the video)

> This consolidates all the Lecture-03/04 banking examples (Customer–Loan total participation, Loan–Payment weak relationship, etc.) into one ER diagram.

### Requirements gathered (assumed)

1. A bank has **Branches**; each branch is located in a city and has a branch **name** (used as identification / primary key).
2. The bank has **Customers**.
3. Customers **hold Accounts**.
4. Customers can take **Loans** (banks earn on the credit/loans they give).
5. A customer is **associated with a Banker** (a relationship manager / loan manager).
6. The bank has **Employees**.
7. Accounts are of **two types: Saving Account & Current Account**.
8. A **Loan is originated by a Branch**; a loan can be **held by multiple customers** (e.g., an education loan held by self + father). A loan has a unique ID, loan amount, and a **Payment Schedule**.
9. **Loan ↔ Payment** is a **weak relationship** (Payment is a weak entity).

---

### Step 1 — Entity Sets

1. `Branch`
2. `Customer`
3. `Employee`
4. `Saving Account`
5. `Current Account`
6. `Loan`
7. `Payment` (a **weak entity**, totally associated with Loan)

> **Bottom-up thinking (Generalization):** `Saving Account` and `Current Account` each have special attributes but are both an _account type_ → **generalize** them into a single `Account` entity (bottom-up = generalization, from Lecture 04).

```
  Generalization applied (Step 1 → refined entity list):

  Saving Account  ──┐
                    ├── IS-A ──► Account  (generalized entity, common attrs here)
  Current Account ──┘

  Final entity list:
  Branch, Customer, Employee, Account, Loan, Payment (weak)
```

---

### Step 2 — Attributes & Types

|Entity|Attributes|Notes|
|---|---|---|
|**Branch**|**Name (PK)**, City, Assets, Liabilities|all single-valued|
|**Customer**|**Customer ID (PK)**, Name, Address **(Composite)**, Contact Number **(Multi-valued)**, Age **(Derived** from DOB**)**, DOB|—|
|**Employee**|**Employee ID (PK)**, Name, Contact Number (single-valued, assumed), Dependent Name **(Multi-valued)**, Start Date, Years of Service **(Derived** from current date − start date**)**|—|
|**Saving Account**|**Account Number (PK)**, Balance, Interest Rate, Daily Withdrawal Limit|—|
|**Current Account**|**Account Number (PK)**, Balance, Transaction Charges, Overdraft Amount|—|
|**Account** (generalized)|**Account Number (PK)**, Balance (common attributes moved up)|overlapping attrs removed from Saving/Current|
|**Loan**|**Loan Number / Loan ID (PK)**, Amount|—|
|**Payment** (weak)|**Payment Number** (partial/weak key)|uses Loan's PK (Loan Number) for identification|

```
  Customer attributes illustrated:

  ┌─────────────────────────────────────────────┐
  │ Customer                                    │
  │                                             │
  │ (Customer_ID)  ← simple, PK (underlined)    │
  │ (Name)         ← simple                     │
  │                                             │
  │ (Address) ─────┬── (Street)                 │
  │   composite    ├── (City)                   │  composite attr
  │                ├── (State)                  │
  │                └── (PinCode)                │
  │                                             │
  │ ((Contact_No)) ← double ellipse, multi-val  │
  │ ····(Age)····  ← dotted ellipse, derived    │
  │ (DOB)          ← stored; Age = today − DOB  │
  └─────────────────────────────────────────────┘
```

---

### Step 3 — Relationships, Cardinality & Participation

|#|Relationship|Cardinality|Participation / Notes|
|---|---|---|---|
|1|**Customer borrows Loan**|**M : N**|one customer can take N loans; one loan can be held by M customers. **Loan side = Total Participation** (if a loan exists, at least one customer took it)|
|2|**Loan originated-by Branch**|**N : 1**|one loan is initiated by exactly one branch; one branch initiates N loans. **Loan side = Total Participation** (a loan must originate from some branch)|
|3|**Loan – Loan Payment**|**1 : N**|a weak relationship; one loan has N payments. **Payment side = Total Participation** (a weak entity exists only with its strong entity)|
|4|**Customer deposits Account**|**M : N**|a customer can deposit in N accounts; an account can be held/deposited by M customers (joint accounts)|
|5|**Customer – Banker**|**N : 1**|N customers can be handled by one relationship manager (an Employee); a customer has one banker|
|6|**Employee managed-by Employee**|**N : 1**|**Unary relationship** (Lecture 03): one employee (manager) manages N employees|

```
  Key relationships at a glance:

  Customer ═══M:N borrows══════════════ Loan
  (partial)                           (total ═══)

  Loan ═══N:1 originated-by═══════════ Branch
  (total ═══)                          (partial)

  Loan ═══1:N loan-payment══════════ ╔═Payment═╗
                                     (total ═══, weak entity)

  Customer ═══M:N deposits═════════════ Account
                                         IS-A
                                   ┌─────┴──────┐
                               SavingAcc   CurrentAcc

  Customer ═══N:1 associated-with══════ Employee (Banker)

  Employee ═══N:1 managed-by══════════ Employee (Unary)
```

---

## 4. Drawing the ER Diagram (what it demonstrates)

Consolidating the 3 steps gives the complete Banking System ER diagram, which beautifully demonstrates:

- **Composite attribute** (Address → Street → composite), **Derived** (Age), **Multi-valued** (Contact/Dependent), **Primary Key** (underlined).
- **Extended ER (Generalization):** Saving Account & Current Account → `IS-A` → `Account` (generalized entity with common attributes; specific attributes stay on the children).
- **Total Participation** (Customer borrows Loan — loan side).
- **Weak Relationship & Weak Entity** (Loan ↔ Payment; Payment has no own primary key → uses Loan Number).
- **Unary Relationship** (Employee managed-by Employee).

```
  Complete Banking ER Diagram (simplified ASCII):
  (double lines ══ = total participation; single lines ── = partial participation)

  Branch attrs: (Name-PK)  (City)  (Assets)  (Liabilities)
                      │
                ┌─────┴──────┐
                │   Branch   │
                └─────┬──────┘
                      │  (partial — a branch may not yet have any loan)
                      │
                 ◇ originated-by   [N:1 — each loan from exactly one branch]
                      │
                      ║  (total — every loan must originate from a branch)
                ┌─────╨──────┐          ╔════════════════╗
                │    Loan    │══════════╣    Payment     ║  (weak entity)
                │ (Loan_ID)  │  ◇loan-  ║ (·Pay_Num·)   ║  ← dotted underline
                │ (Amount)   │  payment ╚════════════════╝
                └─────┬──────┘  [1:N; Payment side = total participation]
                      ║  (total — every loan is borrowed by someone)
                      │
                 ◇ borrows   [M:N — one customer many loans; one loan many customers]
                      │
                      │  (partial — a customer may have no loan)
  Employee attrs:     │                    ┌──────────────────────┐
  (Emp_ID)  (Name)  ┌─┴─────────┐         │       Employee       │◄──┐
  (Contact) (Start) │ Customer  │──────── ◇ associated-with [N:1]│   │
  ((Dep_Name))      │ (Cust_ID) │         │ (Emp_ID) (Name)      │   │ ◇ managed-by
  (···YrsService···)│ (Name)    │         │ ((Dep_Name))         │   │ [N:1 unary —
                    │ (Address*)│         │ (StartDate)          │───┘  manager is
                    │((Contact))│         │ (···YrsOfService···) │      also employee]
                    │ (···Age···│         └──────────────────────┘
                    │ (DOB)     │
                    └─────┬─────┘
                          │  (partial — a customer may not hold an account)
                     ◇ deposits   [M:N — customer many accounts; account many customers]
                          │
                    ┌─────┴──────┐
                    │  Account   │  (generalized entity)
                    │ (AcctNum)  │
                    │ (Balance)  │
                    └─────┬──────┘
                          │ IS-A
              ┌───────────┴──────────────┐
              ▼                          ▼
       Saving Account             Current Account
       (InterestRate)             (TransactionCharges)
       (DailyWithdrawLimit)       (OverdraftAmount)

  Legend:
    *   = composite attribute (e.g., Address → Street, City, State, PinCode)
    (()):  multi-valued attribute (e.g., Contact_No, Dep_Name)
    ···:  derived attribute (e.g., Age from DOB; YrsOfService from StartDate)
    ══:   total participation (double line)
    ──:   partial participation (single line)
    ╔╗/╚╝: weak entity (double rectangle)
    ·key·: partial/discriminator key (dotted underline)
```

---

## 5. Summary & Homework

**Method = 3 steps:** (1) Entities → (2) Attributes + types → (3) Relationships + cardinality + participation. The 3rd step (relationships & constraints) is the hardest and the most important for interviews.

```
  Interview answer template for "design an ER diagram for X":

  1. State your assumptions.
  2. List entity sets.
  3. List attributes per entity (note composite, derived, multi-valued, PK).
  4. List relationships with:
       - cardinality (1:1 / 1:N / N:1 / M:N)
       - participation (total ═══ or partial ───) on each side
       - degree (unary / binary / ternary) and whether weak/strong.
  5. Apply EER if needed (generalize overlapping entities, specialize bloated ones).
```

**Homework (from video):**

1. **Online Delivery System** — entities: Customer, Product, Order, Payment, …; think about mapping cardinality (one customer orders N products, etc.).
2. **University System** — entities: Professor, Student, Course, Department.

> **The only way to learn ER diagrams is to draw many of them** — practice opens your mind to splitting a system into entities and establishing relationships/constraints.

---

> This completes the first 5 lectures (01–05): DBMS basics → architecture/DBA → ER Model → Extended ER features → formulating ER diagrams.