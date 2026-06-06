# Lecture 09 — Normalisation

---

## 1. Functional Dependency (FD)

Before normalization, understand FD.

- Given a table with attributes `A, B, C, D`, an FD like **`A → B`** ("A determines B") means: **if I give you the value of `A`, you can determine the value of `B`.**
- Analogous to a primary key: knowing the PK lets you fetch the rest of the data.
- **`A`** = the left side, a single attribute or a **set of attributes** (two attributes can combine to form a PK).

```
  Functional Dependency — what it means concretely:

  Employee Table:
  +--------+------+------------+
  | EMP_ID | Name | Department |
  +--------+------+------------+
  |  101   | Ali  |     CS     |
  |  102   | Bob  |     IT     |
  +--------+------+------------+

  EMP_ID → Name:
    Give me EMP_ID = 101  -->  I can tell you Name = Ali. Always.
    There is EXACTLY ONE Name for each EMP_ID value.
    (If two employees could share an EMP_ID, this FD would not hold.)

  EMP_ID → Department:
    Give me EMP_ID = 102  -->  I can tell you Department = IT.

  Name → EMP_ID would NOT hold if two people share a name:
    "Ali" could map to EMP_ID 101 or some other ID -> not deterministic.
```

### Terminology
| Term | Meaning |
|------|---------|
| **Determinant** | The left side (`A`) — its value lets you derive another's value |
| **Dependent** | The right side (`B`) — its value depends on `A` |

**Definition:** A functional dependency is a **relationship between the primary-key attribute(s)** of a relation **and another attribute**. (PK or candidate key on the left.)

**Example — Employee table:** `EMP_ID, Name, Department`, PK = `EMP_ID`.
- `EMP_ID → Employee Name` (EMP_ID determines name)
- `EMP_ID → Department`

> Shorthand: FD is written **FD**.

---

## 2. Types of Functional Dependency

### 2.1 Trivial FD
- `A → B` where **`B` is a subset of `A`** (`B ⊆ A`).
- "Trivial" = obvious / ordinary.
- Examples:
  - `{EMP_ID, Name} → EMP_ID` (the dependent is contained in the determinant).
  - `A → A` (A always determines A).
  - `B → B`.

### 2.2 Non-Trivial FD
- `A → B` where **`B` is NOT a subset of `A`**, i.e. `A ∩ B = ∅` (the intersection is empty — they share no attributes).
- Example: `{EMP_ID, Name} → Employee Address` (Address is not a subset of `{EMP_ID, Name}`).
- Example: `EMP_ID → Department` (Department is not a subset of EMP_ID).

**Venn diagrams:**
```
  Trivial FD (B is inside A):         Non-Trivial FD (A and B are disjoint):

  +--------- A ----------+            +---- A ----+    +---- B ----+
  |    +---- B ----+     |            |           |    |           |
  |    |           |     |            |           |    |           |
  |    +-----------+     |            +-----------+    +-----------+
  +----------------------+
  B ⊆ A  -->  A → B is trivial.      A ∩ B = ∅  -->  A → B is non-trivial.
  "Knowing {EMP_ID, Name}, you can   "Knowing EMP_ID, you can determine
  obviously also know EMP_ID alone." Department" — not contained in each other.

  Trivial FDs are logically obvious and always true.
  Non-trivial FDs are the meaningful ones used in normalization.
```

---

## 3. Armstrong's Axioms (FD Rules)

These are the three inference rules for deriving new FDs from existing ones.

### Rule 1 — Reflexivity
- If `A` is a set of attributes and **`B ⊆ A`**, then **`A → B` holds**.
- (Similar to trivial FD.) If `Y ⊆ X`, then `X → Y` — knowing `X`, you can determine `Y`.

### Rule 2 — Augmentation
- If `A → B`, then **adding an attribute to both sides changes nothing**: `AC → BC` (or `XZ → YZ`).
- If `X → Y` holds, then `XZ → YZ` is also a valid FD.

### Rule 3 — Transitivity
- If `A → B` and `B → C`, then **`A → C`**.

```
  Armstrong's Axioms — how each one works:

  Reflexivity:
    {EMP_ID, Name} --> EMP_ID    (EMP_ID is inside the left side -> trivially true)
    Rule: if B ⊆ A, then A → B.

  Augmentation:
    Given: EMP_ID --> Department
    Add attribute "Name" to BOTH sides:
    {EMP_ID, Name} --> {Department, Name}    still holds
    Rule: if A → B, then AC → BC.
    Intuition: adding the same attribute to both sides of a valid FD
    doesn't break it; the new attribute "comes along for the ride".

  Transitivity:
    Given: EMP_ID --> Department
           Department --> HOD
    Therefore: EMP_ID --> HOD    (chain rule)
    Rule: if A → B and B → C, then A → C.
    Intuition: if knowing A tells you B, and knowing B tells you C,
    then knowing A indirectly tells you C.
```

### Worked check — legal vs. illegal FDs
Given relation `A, B, C, D, E` with FDs:
`A → B`, `A → C`, `CD → E`, `B → D`, `E → A`.

- **Is `BC → CD` valid?** → `B → D` is given; augment both sides with `C` → `BC → CD`. **Valid.**
- **Is `EC → A` valid?** → `E → A` is given; augment with `C` → `EC → CA`, so `EC → A`. **Valid.**
- **Is `BD → CD` valid?** → would need `B → C` (remove `D`); no such FD is given → **Illegal FD.**

### Worked proof — Transitivity + Reflexivity
Given `A, B, C, D, E` with FDs:
`A → B`, `A → C`, `CD → E`, `B → D`, `E → A`.

**Prove `CD → AC`:**
1. `CD → E` (given) and `E → A` (given) → by **transitivity**, `CD → A`.
2. `CD → C` (since `C ⊆ CD`, by **reflexivity / trivial FD**).
3. Therefore `CD → A` and `CD → C` → **`CD → AC`. Valid.** ✔

---

## 4. Why Normalize? — To Avoid Redundancy

- **Normalization** is used to **avoid redundancy** in a database (not storing redundant values).
- Redundancy introduces many problems; normalization **optimizes** the DB so redundant values are minimal/none.

### Problems caused by redundant data → 3 Anomalies
Redundant data introduces **three anomalies** (a.k.a. data-dependency anomalies):
1. **Insertion Anomaly**
2. **Deletion Anomaly**
3. **Updation (Modification) Anomaly**

### Example table (un-normalized — has redundancy)
**Student table:** `ID, Name, Age, Branch_Code, Branch_Name, Branch_HOD`
(Mixing Student info and Branch info into one table.)

```
  Un-normalized Student table — redundancy highlighted:

  +----+------+-----+------+-------------+----------+
  | ID | Name | Age | B_Co | Branch_Name | HOD      |
  +----+------+-----+------+-------------+----------+
  |  1 | Avi  |  20 |  CS  |  Computer   | Prof. X  |
  |  2 | Bob  |  21 |  CS  |  Computer   | Prof. X  |  <-- CS data repeated
  |  3 | Sai  |  19 |  IT  |  InfoTech   | Prof. Y  |
  |  4 | Raj  |  22 |  CS  |  Computer   | Prof. X  |  <-- CS data repeated again
  +----+------+-----+------+-------------+----------+

  "Computer Science" branch info (B_Co=CS, Branch_Name=Computer, HOD=Prof. X)
  is stored once per student, not once per branch.
  If 500 CS students exist, this branch data is stored 500 times.
```

### 4.1 Insertion Anomaly
> Certain data cannot be inserted into the DB **without the presence of other data**.
- A new student enrolls but **hasn't chosen a branch yet** → you can't insert the row, or you must insert **NULLs** for the branch columns and update later (double work).
- Or: you add an **IT department** (Branch Code 4) but **no student is yet enrolled** in IT → you can't insert IT without a student, or you insert NULLs.
- Root cause: Student existence and Branch existence are **independent**, but they were merged.

### 4.2 Deletion Anomaly
> Deleting data results in the **unintended loss of other important data**.
- If the **only** Mechanical Engineering student is deleted (passed out), the **Mechanical Engineering branch** disappears from the DB entirely — making it look like the university doesn't teach Mechanical.

```
  Deletion Anomaly:

  Suppose there is only one MECH student:
  +----+------+-----+------+------------+----------+
  |  5 | Zara |  23 | MECH | Mechanical | Prof. Z  |
  +----+------+-----+------+------------+----------+

  When Zara graduates, we DELETE her row.
  Result: the only record of the MECH branch is gone.
  The university still runs Mechanical Engineering, but the DB
  no longer knows this — unintended information loss.
```

### 4.3 Updation (Modification) Anomaly
> A single value update requires **multiple rows to be updated**.
- If the CS-department HOD changes (X → Q), you must update **every** enrolled CS student's row.
- A change required in *one place* must be made in *many places* → if some rows are missed → **data inconsistency**.

### Other downsides
- **DB size increases** (data repeated across rows) → slower searches/queries.

---

## 5. What Normalization Does

- **Decompose** a table into **multiple tables**.
- Keep decomposing until **SRP (Single Responsibility Principle)** is achieved — **one table should represent one single idea / do one job**.

### Quick fix of the example (intuition)
- **Table 1:** `ID, Name, Age, Branch_Code`
- **Table 2 (Branch Info):** `Branch_Code, Branch_Name, HOD_Name`

Now: no insertion anomaly (add IT department without students), no deletion anomaly, no updation anomaly. HOD change → update **one** row in Branch Info.

```
  After decomposition — all three anomalies gone:

  Student Table:                         Branch Table:
  +----+------+-----+------+             +------+-------------+----------+
  | ID | Name | Age | B_Co |             | B_Co | Branch_Name | HOD      |
  +----+------+-----+------+             +------+-------------+----------+
  |  1 | Avi  |  20 |  CS  |----------->|  CS  |  Computer   | Prof. X  |
  |  2 | Bob  |  21 |  CS  |            |  IT  |  InfoTech   | Prof. Y  |
  |  3 | Sai  |  19 |  IT  |----------->+------+-------------+----------+
  +----+------+-----+------+
  B_Co is FK in Student -> Branch.B_Co

  Insertion fix: add MECH branch to Branch table even with no students yet.
  Deletion fix:  delete a student row; Branch table is untouched.
  Updation fix:  HOD change -> update ONE row in Branch table only.
```

---

## 6. Normal Forms

Decompose progressively, each form adding more restrictions and further optimizing the DB.

```
  Normal Form progression — each level includes all previous requirements:

  Un-normalized  (multi-valued cells allowed, redundancy allowed)
       |
       v
     1NF   -- no multi-valued cells; every cell holds one atomic value
       |
       v
     2NF   -- 1NF + no partial dependency
       |           (non-prime attrs must depend on the WHOLE PK, not part of it)
       v
     3NF   -- 2NF + no transitive dependency
       |           (non-prime attr must not determine another non-prime attr)
       v
    BCNF   -- 3NF + every determinant in every FD must be a super key
```

---

### 6.1 First Normal Form (1NF)
- **Every cell of a relation must contain an atomic value.**
- The relation **must not have multi-valued attributes.**

**Example — Employee table (NOT in 1NF):**
| ID | Name | Phone Number |
|----|------|--------------|
| 1 | A | 88..., (multiple) |

→ Convert to 1NF by giving each phone its own row (atomic value):
| ID | Name | Phone Number |
|----|------|--------------|
| 1 | A | 88... |
| 1 | A | (other) |

```
  1NF violation and fix:

  NOT in 1NF:                            In 1NF:
  +----+------+------------------+       +----+------+-----------+
  | ID | Name | Phone            |       | ID | Name | Phone     |
  +----+------+------------------+       +----+------+-----------+
  |  1 |  A   | 88xxx, 99xxx    |  -->  |  1 |  A   | 88xxx     |
  +----+------+------------------+       |  1 |  A   | 99xxx     |
                                         +----+------+-----------+

  The Phone cell had two values in it ("88xxx, 99xxx").
  That is not atomic — it's a list inside one cell.
  Fix: one phone per row. Each cell now holds exactly one value.

  Note: 1NF introduces row repetition (ID=1 appears twice).
  1NF does NOT eliminate redundancy — that is handled by 2NF and 3NF.
  1NF's only job is atomicity.
```

> 1NF **only** requires atomic values; it does **not** care about the data repetition this may introduce. (This is the same idea as decomposing a multi-valued attribute when going from ER → relational model.)

---

### 6.2 Second Normal Form (2NF)
**Conditions:**
1. Relation is in **1NF** (all atomic values).
2. **No partial dependency** — every non-prime attribute must be **fully dependent** on the (whole) primary key, not on a *part* of it.

**Partial dependency definition:** A **non-prime attribute** (one not part of the candidate/primary key) depends on **part of** a (composite) primary key.

**Why partial dependency is a problem:** If PK = `{A, B}` (composite) and `B → C` (a part of the PK alone determines a non-prime attribute C), then the value of C is repeated in every row that shares the same value of B — regardless of A. This causes the same redundancy, insertion anomalies, deletion anomalies, and updation anomalies as in the un-normalized examples above. 2NF eliminates this by requiring every non-prime attribute to depend on the *whole* composite PK, not just a part of it.

```
  What "partial dependency" means:

  A non-prime attribute (not part of the PK) that can be determined
  by ONLY A PART of the composite PK — not the full PK.

  Example: Student_Project table
  PK = {Student_ID, Project_ID}   (both together form the primary key)
  Non-prime attributes: Student_Name, Project_Name

  Student_ID  --> Student_Name    PARTIAL DEPENDENCY
  (Student_Name is determined by only Student_ID, one half of the PK.
   We don't need Project_ID to know someone's name.)

  Project_ID  --> Project_Name    PARTIAL DEPENDENCY
  (Project_Name is determined by only Project_ID, the other half.)

  {Student_ID, Project_ID} --> Student_Name    is NOT a full dependency
  because Project_ID is unnecessary for determining Student_Name.

  Full dependency requires ALL parts of the composite PK to be needed:
  {Student_ID, Project_ID} --> Grade    could be a full dependency
  (the grade of a student ON a specific project needs BOTH IDs).
```

**Conversion example:** Relation `A, B, C, D` with PK `{A, B}`, FD `B → C` (partial dependency), and `{A, B} → D` (full dependency).
Decompose into:
- **R1:** `A, B, D` (PK = `{A, B}`), removing the attribute `C` that was partially determined by only `B`.
- **R2:** `B, C` (PK = `B`), capturing the partial dependency `B → C` in its own table.

**Real-life example — Student Project table:**
| Student_ID | Project_ID | Student_Name | Project_Name |

- PK (composite) = `{Student_ID, Project_ID}`; prime = those two; non-prime = `Student_Name, Project_Name`.
- FDs: `Student_ID → Student_Name`, `Project_ID → Project_Name` → **both are partial dependencies** (a non-prime attr determined by a *part* of the PK).

**Decompose (2NF):**
- **Student table:** `Student_ID (PK), Student_Name` → `Student_ID → Student_Name` (full dependency)
- **Project table:** `Project_ID (PK), Project_Name` → `Project_ID → Project_Name` (full dependency)
- **Enrollment table:** `Student_ID (FK, part of PK), Project_ID (FK, part of PK)` — retains the original composite PK relationship between Student and Project, preserving which student works on which project.

> 2NF **completely removes partial dependencies.**

---

### 6.3 Third Normal Form (3NF)
**Conditions:**
1. Relation is in **2NF**.
2. **No transitive dependency.** The formal rule: for every non-trivial FD `α → β`, at least one of these must be true: (a) `α` is a super key, OR (b) every attribute in `β` is a **prime attribute** (part of some candidate key). The most common violation — and the one to focus on — is a **non-prime attribute determining another non-prime attribute** (transitive dependency).

**Example:** Relation `A, B, C`, PK = `A`, FDs: `A → B`, `B → C`.
- `A → B` and `B → C` ⇒ `A → C` (transitive). So why keep `B → C`? Because `B` (non-prime) determines `C` (non-prime) → **transitive dependency** → causes **redundancy** (repeated data).

```
  Transitive dependency — what it looks like and why it causes problems:

  Table: {Student_ID, Zip_Code, City}   PK = Student_ID
  FDs:
    Student_ID --> Zip_Code   (direct: student's zip is determined by student)
    Zip_Code   --> City       (transitive: zip determines city)
    Student_ID --> City       (by transitivity, but also means every student
                               in the same zip stores the same City value)

  +------------+----------+--------+
  | Student_ID | Zip_Code | City   |
  +------------+----------+--------+
  |    101     |  110001  | Delhi  |
  |    102     |  110001  | Delhi  |  <-- "Delhi" repeated! Redundancy.
  |    103     |  400001  | Mumbai |
  +------------+----------+--------+

  The problem: Zip_Code (non-prime) determines City (non-prime).
  This is the transitive dependency. City is stored once per student,
  but it really belongs to the Zip_Code, not to the student directly.
  If Delhi changes its name, every student row in zip 110001 must update.
```

**Conversion (3NF):** Decompose:
- **R1:** `A, B` (PK = `A`)
- **R2:** `B, C` (PK = `B`)

→ Removes the transitive dependency and the redundancy.

---

### 6.4 Boyce-Codd Normal Form (BCNF)
- Full form: **Boyce-Codd Normal Form**. A **stronger version of 3NF**.

**Conditions:**
1. Relation is in **3NF**.
2. For every non-trivial FD `A → B`, **`A` must be a super key.**

> Progression:
> - 3NF removed non-prime → non-prime determination (but was silent about non-super-keys determining prime attributes).
> - BCNF closes that gap: **every determinant in every non-trivial FD must be a superkey**, regardless of whether the right-hand side is prime or non-prime. The most common violation is a non-super-key attribute determining a prime attribute (as in the example below), which 3NF does not catch.

```
  The gap between 3NF and BCNF:

  3NF says: a non-prime attribute must not determine another non-prime attribute.
  But 3NF is SILENT about a non-prime attribute determining a PRIME attribute.

  BCNF closes that gap:
    For EVERY FD  X --> Y  in the table,
    X must be a super key.
    (If X is not a super key, the FD violates BCNF regardless of what Y is.)

  3NF forbids: non-prime  ---> non-prime
  BCNF forbids: non-super-key ---> anything  (including prime attributes)
```

**Example — `Student_ID, Subject, Professor`:**
- Constraints: one student enrolls in multiple subjects; **each subject** for a student has an assigned professor; **one professor teaches only one subject**.
- PK (composite) = `{Student_ID, Subject}`.
- FDs:
  - `{Student_ID, Subject} → Professor`
  - `Professor → Subject` (since one professor teaches exactly one subject)
- This is in 1NF, 2NF (no partial dependency), and 3NF — **but** in `Professor → Subject`, **`Subject` is a prime attribute** and `Professor` is **not a super key** → **violates BCNF.**

```
  Why the BCNF example satisfies 3NF but violates BCNF:

  Table: {Student_ID, Subject, Professor}
  PK = {Student_ID, Subject}
  Prime attributes:     Student_ID, Subject
  Non-prime attributes: Professor

  FD: {Student_ID, Subject} --> Professor     LHS is the full PK = super key. OK.
  FD: Professor --> Subject                   LHS is Professor (non-prime, NOT a super key).
                                              RHS is Subject (PRIME attribute).

  3NF check: "does a non-prime determine a non-prime?" 
    Professor (non-prime) --> Subject (PRIME) -- the RHS is prime, so 3NF does NOT catch this.
    This passes 3NF! (3NF only forbids non-prime -> non-prime)

  BCNF check: "is every LHS of every FD a super key?"
    Professor is NOT a super key (Professor alone cannot uniquely identify a tuple,
    since a professor could teach the same subject to many students).
    --> BCNF VIOLATED.

  The redundancy: Prof. PJ always teaches Java. Every row with PJ stores Subject=Java.
  +------------+---------+-----------+
  | Student_ID | Subject | Professor |
  +------------+---------+-----------+
  |    101     |  Java   |    PJ     |
  |    102     |  Java   |    PJ     |  <-- PJ -> Java repeated for every student
  +------------+---------+-----------+
```

**Conversion (BCNF):** Decompose by moving the violating FD (`Professor → Subject`) into its own table:
- **Professor table:** `Professor (PK), Subject` — captures the FD `Professor → Subject` directly. Professor is now the sole key of this table (a super key), so BCNF is satisfied here.
  | Professor | Subject |
  |-----------|---------|
  | PJ        | Java    |
  | PC        | C++     |
- **Student_Professor table:** `Student_ID (part of PK), Professor (FK → Professor table, part of PK)` — retains which student is assigned which professor.
  | Student_ID | Professor |
  |------------|-----------|
  | 101        | PJ        |
  | 102        | PJ        |

→ In both resulting tables every determinant in every non-trivial FD is a super key (Professor is the PK of the Professor table; {Student_ID, Professor} is the PK of the Student_Professor table) → BCNF satisfied. Subject can be retrieved by joining through Professor.

---

## 7. Normal Forms — Consolidated Summary

For a dependency `α → β`:

| Form | Rule |
|------|------|
| **1NF** | All cell values atomic; no multi-valued attributes. |
| **2NF** | 1NF + **no partial dependency** (`β` must not depend on *part* of a composite PK). |
| **3NF** | 2NF + for every non-trivial FD `α → β`, **at least one** of the following must hold: (a) `α` is a super key, OR (b) `β` is a prime attribute (part of some candidate key). In practice the main violation to eliminate is a **non-prime attribute determining another non-prime attribute** (transitive dependency). |
| **BCNF** | 3NF + for every non-trivial `α → β`, `α` **must** be a super key — no exceptions. Stricter than 3NF because 3NF allowed non-super-key `α` as long as `β` was prime; BCNF eliminates that loophole. |

---

## 8. Advantages of Normalization

- **Removes data redundancy** → no insertion/deletion/updation **anomalies**.
- Overall **well-structured DB**.
- **No data inconsistency.**
- **Faster, optimized DB** — takes less space; queries run optimized.

**Next lecture:** Transactions & ACID Properties.
