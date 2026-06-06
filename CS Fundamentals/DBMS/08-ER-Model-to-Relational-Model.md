# Lecture 08 — Transformation from ER Model to Relational Model

---

## 1. Context

- **DB design = ER Model (→ ER Diagram) → Relational Model (→ tables / conceptual schema).**
- The relational model gives many **relation schemas**; relationships between them are shown using **foreign keys** (referential constraints).
- **Goal:** Given an ER diagram, reduce it to a relational model — how many tables, which primary keys, which foreign keys, and how relationships are shown.
- Uses the **Banking System ER diagram from Lecture 05** (which had weak entities, weak relationships, generalization, and unary relationships).

---

## 2. Converting ER Notations into Relations (Tables)

### 2.1 Strong Entity
- Each **strong entity → its own individual table**.
- Each **single-valued attribute → a column**.
- **Table name = entity name.**
- **Primary key** = the same PK given to that entity in the ER diagram.
- A **foreign key** is added if the entity establishes a relationship with another.

**Example — `Loan` (strong entity):**
| Loan Number (PK) | Amount |

### 2.2 Weak Entity
- A weak entity has no existence of its own; it **totally participates / depends on a strong entity**.
- Reduce it to a table, then add the **PK of the owner (strong) entity** into it.
- **Primary Key = (PK of strong entity) + (discriminator of weak entity)** — a composite key.
- The strong entity's PK inside the weak-entity table is also a **foreign key**.

**Example — `Payment` (weak entity, owner = `Loan`):**
| Loan Number (FK, part of PK) | Payment Number (discriminator, part of PK) | ...other attrs |

```
  Why the composite PK for a weak entity?

  The discriminator (Payment Number) only distinguishes payments
  WITHIN one loan — it is not globally unique across all loans:

    Loan 1 has: Payment 1, Payment 2
    Loan 2 has: Payment 1, Payment 2   <-- "Payment 1" appears for both loans!

  Payment Number alone as PK would create duplicate PKs -> invalid.

  Combined (Loan_Number, Payment_Number) IS globally unique:
    (1, 1), (1, 2), (2, 1), (2, 2)  -- all distinct -> valid composite PK

  The Loan_Number inside the Payment table is simultaneously:
    - Part of Payment's PK (composite key component)
    - A foreign key referencing the Loan table
```

> The discriminator (`Payment Number`) cannot be a PK by itself because the weak entity depends on the strong entity → combine **strong-entity PK + discriminator**.

### 2.3 Single-Valued Attribute
- Simply acts as **one column** in the relation.

### 2.4 Composite Attribute
- **Break it into separate (component) attributes** when building the table.

**Example — `Customer` with composite `Address` (and `Street` itself composite):**
`Address` had 5 components → flatten:
| Customer Name | Address_City | Address_State | Address_PinCode | Address_StreetNumber | Address_StreetName |

> Store only the **leaf components** as separate columns.

```
  Composite attribute flattening:

  In the ER diagram, Address is one composite oval with sub-ovals inside it.
  In the relational table, the composite oval itself disappears;
  only its innermost (leaf) components become columns.

  Address (composite — NOT a column itself)
  ├── City          --> column: Address_City
  ├── State         --> column: Address_State
  ├── PinCode       --> column: Address_PinCode
  └── Street (also composite — NOT a column itself)
      ├── Number    --> column: Address_StreetNumber
      └── Name      --> column: Address_StreetName

  Result: 5 columns, no "Address" column.
  The nesting is flattened; only leaf values are stored.
```

### 2.5 Multi-Valued Attribute → New Table
- A multi-valued attribute is **moved into a NEW table (relation)**.
- The new table's name = the multi-valued attribute's name.
- It holds: the **owner entity's PK as a foreign key** + a **column for the value**.
- **Primary Key of the new table = (FK) + (the multi-valued column)** — composite.

**Example — `Dependent Name` (multi-valued on Employee):**
A single Employee can have multiple dependents (father, mother, wife...). Don't repeat all employee attributes; make a new table:

**Table: `Dependent_Name`**
| EMP_ID (FK → Employee, part of PK) | D_Name (part of PK) |
|------------------------------------|---------------------|
| 1 | ... |
| 1 | ... |

- `EMP_ID` is a **foreign key** (PK of Employee table) → lets you reference the Employee table.
- PK = `(EMP_ID, D_Name)`.

```
  Why a multi-valued attribute must become its own table:

  If we tried to keep dependents inside the Employee table:
  +--------+------+----------+----------+----------+
  | EMP_ID | Name | Dep_1    | Dep_2    | Dep_3    |
  +--------+------+----------+----------+----------+
  |   1    |  A   | Father   | Mother   |  NULL    |
  |   2    |  B   | Wife     | NULL     |  NULL    |
  +--------+------+----------+----------+----------+
  Problems: (1) How many Dep columns to create? Unknown.
            (2) Most columns are NULL for most employees.
            (3) Values are not atomic — violates relational model property 2.

  Correct approach — separate table:
  +--------+--------+
  | EMP_ID | D_Name |   PK = (EMP_ID, D_Name)
  +--------+--------+
  |   1    | Father |   EMP_ID is FK -> Employee.EMP_ID
  |   1    | Mother |   Each dependent is one row. Atomic. No NULLs.
  |   2    | Wife   |   To get all dependents of Employee 1:
  +--------+--------+   SELECT * WHERE EMP_ID = 1  -> returns 2 rows.

  PK = (EMP_ID, D_Name): one employee can't have two dependents
  with the exact same name, so this pair is unique.
```

### 2.6 Derived Attribute
- **Not stored in tables.** Used only as an ER-diagram notation.
- It can be computed directly via the API (e.g., `Age` is derived from DOB and current date), so there is no place for it in the table.

---

## 3. Generalization → Two Methods

Example: generalized entity `Account`, specialized into `Current Account` and `Saving Account`.

### Method 1 — Table for higher-level entity + tables for lower-level entities
- Create a table for the **parent (higher-level)** entity **and** for each **child (lower-level)** entity.

**Table 1: Account**
| Account Number (PK) | Balance |

**Table 2: Saving Account** (PK = Account Number from parent)
| Account Number (PK) | Interest | Daily Withdrawal Limit |

**Table 3: Current Account** (PK = Account Number from parent)
| Account Number (PK) | Overdraft Amount | Per-Transaction Charges |

- Lower-level tables **reuse the generalized entity's PK** as their PK.

### Method 2 — Drop the parent table; keep only lower-level tables
- Remove the `Account` table.
- Make tables only for `Saving Account` and `Current Account`.
- `Account Number` is the PK in both anyway; **push down the parent's attributes (e.g., `Balance`) into both child tables.**

```
  Method 1 (parent + children):           Method 2 (children only):

  Account                                 (no Account table)
  +--------------+--------+
  | Acct_No (PK) | Balance|
  +--------------+--------+               Saving_Account
       |                                  +--------------+--------+----------+
       +--> Saving_Account                | Acct_No (PK) | Balance| Interest |
       |    +--------------+----------+   +--------------+--------+----------+
       |    | Acct_No (PK) | Interest |
       |    +--------------+----------+   Current_Account
       |                                  +--------------+--------+-----+
       +--> Current_Account               | Acct_No (PK) | Balance| ... |
            +--------------+------+       +--------------+--------+-----+
            | Acct_No (PK) | ...  |
            +--------------+------+       Balance is duplicated into both child
                                          tables (stored twice for overlapping accts)
  Balance stored ONCE in Account.
```

### Drawbacks of Method 2

**(a) Overlapping generalization** → causes **redundancy**.
- If an account is **both** Saving **and** Current (same account number, both features), there is no generalized entity to store shared data.
- You must store an entry in **both** Saving and Current tables, and the shared `Balance` (₹10) gets **stored twice** → redundancy.
- **Method 1 avoids this:** `Balance` is stored only once (in the Account table); child tables don't store balance.

**(b) Incomplete (non-total) generalization** → some accounts can't be represented.
- If an account is **neither** Saving **nor** Current (a third type not stored), Method 2 has **no table** to store it (no generalized `Account` table exists).
- "Such an account could not be represented with the second method."

> **Verdict:** Method 2 looks simpler (only 2 tables), but it **breaks** when generalization is overlapping (a record belongs to multiple sub-classes) or incomplete (a record belongs to neither). **Method 1** (keeping the parent table) handles both cases correctly because shared attributes live in one place and every sub-type record has a parent row. Use Method 2 only when generalization is total and disjoint (every entity belongs to exactly one sub-class, never two).

---

## 4. Aggregation

Recall (Lecture 04): a **ternary relationship** among `Job`, `Employee`, `Branch` was aggregated into a single `Work On` entity, and a relationship was established between `Work On` and `Manager` (a manager *manages* that combination).

### Reducing aggregation to a table
- Take the **relationship between the aggregated entity and the normal (strong) entity** → make it a **table**.
- Columns = **primary keys of all participating entities**: `Manager ID`, `EMP ID`, `Job ID`, `Branch ID`.
- **Primary Key = composite** of all these PKs.
- Also **add any descriptive attribute** present on the relationship as a column of this table.

```
  Aggregation reduction:

  From the ER diagram (Lecture 04):
    Entities: Employee, Job, Branch are in a ternary relationship "Work_On".
    That entire "Work_On" block is then related to Manager via "Manages".

  To convert to a table, we represent the "Manages" relationship:
    Table: Manages
    +-----------+--------+--------+-----------+
    | Mgr_ID    | EMP_ID | Job_ID | Branch_ID |
    +-----------+--------+--------+-----------+
    PK = (Mgr_ID, EMP_ID, Job_ID, Branch_ID)   composite of all four PKs

    Each column is the PK of one of the participating entities.
    Together they uniquely identify one "manages" instance.
```

---

## 5. Unary (Recursive) Relationships

A relationship of an entity **with itself**. Three sub-types:

### 5.1 Unary 1:N
**Example — `Employee MANAGES Employee` (1:N).**
- In the **Employee** table, add a column that is a **foreign key referencing the Employee table's own PK** (the manager's Employee ID).

**Employee table:**
| EMP_ID (PK) | ... | Manager_ID (FK → Employee.EMP_ID) |
|-------------|-----|-----------------------------------|
| 201 | ... | 205 |
| 202 | ... | 205 |

- One manager (205) manages multiple employees → **1:N**.
- `Manager_ID` **can be NULL** (e.g., a CEO has no manager).

```
  Unary 1:N — self-referencing FK in the SAME table:

  Employee table:
  +---------+------+------------+
  | EMP_ID  | Name | Manager_ID |  <-- FK that points to EMP_ID in the SAME table
  +---------+------+------------+
  |   201   | Ali  |    205     |  Ali's manager is employee 205
  |   202   | Bob  |    205     |  Bob's manager is also 205
  |   205   | Sam  |   NULL     |  Sam is the top manager; no manager above him
  +---------+------+------------+

  Manager_ID references Employee.EMP_ID (same table, same column).
  NULL is allowed because the top-level employee has no manager.
  One manager (205) appears in Manager_ID for multiple rows -> 1:N confirmed.
```

### 5.2 Unary 1:1
**Example — `Person MARRIED TO Person` (monogamy, 1:1).**

**Person table:**
| ID (PK) | Name | Spouse_ID (FK → Person.ID) |

- `Spouse_ID` behaves as a foreign key referencing the same table.

### 5.3 Unary M:N → New (relationship) Table
**Example — `Course PREREQUISITE Course` (M:N).**
- A course can require many prerequisite courses, and be a prerequisite for many → many-to-many.

**Course table:**
| ID (PK) | Title |

**Prerequisite table** (named after the relationship):
| ID (FK → Course.ID) | Prereq_Course_ID (FK → Course.ID) |

- **Both** columns are foreign keys referencing the Course table's PK.
- **Primary Key = composite** of both columns.

```
  Unary M:N — why a separate table is needed:

  Course table:
  +----+---------+
  | ID | Title   |
  +----+---------+
  |  1 | Math    |
  |  2 | Calculus|
  |  3 | Physics |
  +----+---------+

  Facts: Physics requires Math AND Calculus.
         Math requires Calculus.

  Prerequisite table (new relationship table):
  +----+-----------------+
  | ID | Prereq_Course_ID|   PK = (ID, Prereq_Course_ID)
  +----+-----------------+
  |  3 |        1        |   Physics requires Math
  |  3 |        2        |   Physics requires Calculus
  |  1 |        2        |   Math requires Calculus
  +----+-----------------+

  Both columns are FK -> Course.ID (pointing to the SAME table).
  A new table is needed because you can't store a many-to-many
  relationship as a single column inside Course.
```

---

## 6. Worked Example — Reducing the Facebook ER Diagram (Lecture 06) to a Relational Model

Applying all the above rules to the Facebook ERD → **9 tables (relations)**:

### Table 1: User Profile
| User Name (PK) | Password | DOB | ... | (single-valued attrs) |

### Table 2: User Profile Email (multi-valued `Email` → new table)
| User Name (FK, part of PK) | Email (part of PK) |
- Composite PK = `(User Name, Email)`.

### Table 3: User Profile Contact (multi-valued `Contact` → new table)
| User Name (FK, part of PK) | Contact (part of PK) |

### Table 4: Friendship (unary M:N on User Profile → new table)
| Profile_Requested (FK → UserProfile.UserName) | Profile_Accepted (FK → UserProfile.UserName) |
- Both are foreign keys (each is some user name).
- **Compound Key** (two FKs combined) → serves as PK.

### Table 5: Post Like (strong entity)
| Post Like ID (PK) | Time Stamp | Post_ID (FK → UserPost) | User_Name (FK → UserProfile) |
- `Post_ID` FK shows the **HAS** relationship (User Post has Post Like).
- `User_Name` FK shows the **CAN/like** relationship (a user liked the post).

### Table 6: User Post (strong entity)
| Post ID (PK) | (content attrs) | User_Name (FK → UserProfile) |
- `User_Name` FK shows the **Posts** relationship (linkage between User Post and User Profile via `User Name`).
- Note: `Image` and `Video` were **multi-valued** → moved to their own tables (below).

### Table 7: User Post Image (multi-valued `Image` → new table)
| Post_ID (FK → UserPost, part of PK) | Image_URL (part of PK) |
- Each image has its own URL; combined → composite PK.

### Table 8: User Post Video (multi-valued `Video` → new table)
| Post_ID (FK → UserPost, part of PK) | Video_URL (part of PK) |

### Table 9: Post Comment
| Post Comment ID (PK) | Text Content | Time Stamp | User_Name (FK → UserProfile) | Post_ID (FK → UserPost) |
- `User_Name` FK → **Comments** + **HAS** relationships (the user profile that commented).
- `Post_ID` FK → links the comment to its post.
- Two foreign keys here.

```
  Why 9 tables? Count the sources:

  Original 4 entities -> 4 tables:
    T1: User Profile
    T5: Post Like       (strong entity)
    T6: User Post       (strong entity)
    T9: Post Comment    (strong entity)

  Multi-valued attributes -> +4 new tables:
    T2: User_Profile_Email    (Email is multi-valued on User Profile)
    T3: User_Profile_Contact  (Contact is multi-valued on User Profile)
    T7: User_Post_Image       (Images is multi-valued on User Post)
    T8: User_Post_Video       (Videos is multi-valued on User Post)

  Unary M:N relationship -> +1 new table:
    T4: Friendship            (M:N self-relationship on User Profile)

  Total: 4 + 4 + 1 = 9 tables.

  Relationships (Posts, Can Like, Comments, Has x2) are captured
  NOT as separate tables, but as FK columns inside the entity tables:
    T6 (User Post) carries User_Name FK  -> represents "Posts"
    T5 (Post Like) carries User_Name FK  -> represents "Can Like"
    T9 (Post Comment) carries User_Name FK -> represents "Comments"
    T5 (Post Like) carries Post_ID FK    -> represents "Has" (Post->Like)
    T9 (Post Comment) carries Post_ID FK -> represents "Has" (Post->Comment)
```

> **Result:** 9 relations. These can be directly implemented in an RDBMS (e.g., MySQL) using **SQL `CREATE TABLE` commands**.

---

## 7. Summary

| ER Construct | Reduction to Relational Model |
|--------------|-------------------------------|
| Strong Entity | Own table; PK preserved |
| Weak Entity | Own table; PK = owner PK + discriminator; owner PK is FK |
| Single-valued attribute | Column |
| Composite attribute | Split into leaf-component columns |
| Multi-valued attribute | New table (owner PK as FK + value); composite PK |
| Derived attribute | Not stored (computed) |
| Generalization | Method 1 (parent + child tables) or Method 2 (only child tables) |
| Aggregation | Table of all participating PKs; composite PK |
| Unary 1:N | Self-referencing FK column |
| Unary 1:1 | Self-referencing FK column (spouse_id) |
| Unary M:N | New relationship table with two self-referencing FKs; composite PK |

**Next lecture:** Normalization.
