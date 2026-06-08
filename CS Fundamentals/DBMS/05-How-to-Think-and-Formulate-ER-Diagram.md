## 0. ER Diagram Construction

- ER Diagram is used to create the **Conceptual Schema**.
- It acts as a **logical blueprint** before converting the design into relational tables.
- Previous topics: Entities, Relationships, Mapping Cardinality, Participation Constraints, ER Notations.
- This topic: **Systematic steps for building an ER Diagram** for any application (university, school, banking, etc.).

```
ER Diagram (Conceptual Schema)
            ↓
Relational Tables(MySQL, PostgreSQL, etc.)
```

## 1. The 3 Steps to Make an ER Diagram
### Step 1 — Identify Entity Sets
Which entities / *entity sets* exist in the system.
### Step 2 — Identify Attributes (and their types)
For each entity, decide *attributes* and their **types** (single-valued, multi-valued, derived, composite) — and the **Primary Key**.
### Step 3 — Identify Relationships & Constraints (MOST IMPORTANT)
- Which relationships to establish (strong / weak).
- **Degree** of the relationship.
- **Mapping Cardinality** (1:1, 1:N, N:1, M:N).
- **Participation Constraint** (Total vs Partial).
---
## 2. Before designing: Requirement Specification
Before building the ER model, **collect DB requirements** — what the DB user/stakeholders want (separate customer data, a loan DB, an accounts DB, etc.). This is **Requirement Engineering**: interview the DB user and stakeholders to remove ambiguity.
> **In an interview** you don't have time for full requirement engineering, so you **assume** many requirements yourself and build the diagram with those assumptions.

```
  Real world:   Requirement Engineering  →  clear requirements  →  ER Design
  Interview:    State your assumptions upfront  →  then design directly
                ("I'm assuming a customer can hold multiple accounts ...")
```
---
## 3. Worked Example — Banking System

> This consolidates the Lecture-03/04 banking examples (Customer–Loan total participation, Loan–Payment weak relationship, etc.) into one ER diagram.

### Requirements gathered (assumed)
1. A bank has **Branches**; each branch is located in a city and has a branch **name** (used as identification / primary key).
2. The bank has **Customers**.
3. Customers **hold Accounts**.
4. Customers can take **Loans**.
5. A customer is **associated with a Banker** (a relationship manager / loan manager).
6. The bank has **Employees**.
7. Accounts are of **two types: Saving Account & Current Account**.
8. A **Loan is originated by a Branch**; a loan can be **held by multiple customers** (e.g., an education loan held by self + father). A loan has a unique ID, loan amount, and a **Payment Schedule**.
9. **Loan ↔ Payment** is a **weak relationship** (Payment is a weak entity).
---
### Step 1 — Entity Sets
1. `Branch`
2. `Customer`
3. `Employee`
4. `Saving Account`
5. `Current Account`
6. `Loan`
7. `Payment` (a **weak entity**, totally associated with Loan)

> **Bottom-up thinking (Generalization):** `Saving Account` and `Current Account` each have special attributes but are both an *account type* → *generalize* them into a single `Account` entity.

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

| Entity | Attributes | Notes |
|--------|-----------|-------|
| **Branch** | **Name (PK)**, City, Assets, Liabilities | all single-valued |
| **Customer** | **Customer ID (PK)**, Name, Address **(Composite)**, Contact Number **(Multi-valued)**, Age **(Derived** from DOB**)**, DOB | — |
| **Employee** | **Employee ID (PK)**, Name, Contact Number (single-valued, assumed), Dependent Name **(Multi-valued)**, Start Date, Years of Service **(Derived** from current date − start date**)** | — |
| **Saving Account** | **Account Number (PK)**, Balance, Interest Rate, Daily Withdrawal Limit | — |
| **Current Account** | **Account Number (PK)**, Balance, Transaction Charges, Overdraft Amount | — |
| **Account** (generalized) | **Account Number (PK)**, Balance (common attributes moved up) | overlapping attrs removed from Saving/Current |
| **Loan** | **Loan Number / Loan ID (PK)**, Amount | — |
| **Payment** (weak) | **Payment Number** (partial/weak key) | uses Loan's PK (Loan Number) for identification |

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

| # | Relationship | Cardinality | Participation / Notes |
|---|--------------|-------------|------------------------|
| 1 | **Customer borrows Loan** | **M : N** | one customer can take N loans; one loan can be held by M customers. **Loan side = Total Participation** (if a loan exists, at least one customer took it) |
| 2 | **Loan originated-by Branch** | **N : 1** | one loan is initiated by exactly one branch; one branch initiates N loans. **Loan side = Total Participation** (a loan must originate from some branch) |
| 3 | **Loan – Loan Payment** | **1 : N** | a weak relationship; one loan has N payments. **Payment side = Total Participation** (a weak entity exists only with its strong entity) |
| 4 | **Customer deposits Account** | **M : N** | a customer can deposit in N accounts; an account can be held/deposited by M customers (joint accounts) |
| 5 | **Customer – Banker** | **N : 1** | N customers can be handled by one relationship manager (an Employee); a customer has one banker |
| 6 | **Employee managed-by Employee** | **N : 1** | **Unary relationship** (Lecture 03): one employee (manager) manages N employees |

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
Consolidating the 3 steps gives the complete Banking System ER diagram, which demonstrates:
- **Composite attribute** (Address → Street → composite), **Derived** (Age), **Multi-valued** (Contact/Dependent), **Primary Key** (underlined).
- **Extended ER (Generalization):** Saving Account & Current Account → `IS-A` → `Account` (generalized entity with common attributes; specific attributes stay on the children).
- **Total Participation** (Customer borrows Loan — loan side).
- **Weak Relationship & Weak Entity** (Loan ↔ Payment; Payment has no own primary key → uses Loan Number).
- **Unary Relationship** (Employee managed-by Employee).
![[Pasted image 20260606154853.png]]
---
