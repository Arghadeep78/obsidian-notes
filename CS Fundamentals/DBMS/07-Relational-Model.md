## 1. What is the Relational Model?
- The Relational Model is a **type of data model** (like the ER model).
- It is used to:
  - **Describe data**
  - **Describe relationships** within the data
  - **Define data consistency constraints**
- Core idea: data is represented in **tabular form (tables)**.
### Table = Relation
- A **table IS a relation** in the relational model. Table and relation are the same thing.
### Example: Customer data
Attributes: `Customer ID`, `Name`, `Address`, `Contact`.

| Customer ID | Name    | Address | Contact |
| ----------- | ------- | ------- | ------- |
| 1           | Lakshya | ...     | ...     |
| 2           | Raj     | ...     | ...     |

- **Columns** = the attributes.
- A **row = a tuple** → each tuple represents **one customer**.
- A set of values in a row together **define one customer** (one real-world entity / relationship of values).
---
## 2. Terminology

| Term                   | Meaning                                        | In the example           |
| ---------------------- | ---------------------------------------------- | ------------------------ |
| **Relation**           | The table itself                               | Customer table           |
| **Tuple**              | A single row / one data point                  | One customer             |
| **Attribute**          | A column                                       | Customer ID, Name, ...   |
| **Degree of Relation** | **Number of attributes (columns)**             | 4                        |
| *Cardinality*          | number of tuples (rows) in a relation (table). | 2 (two unique customers) |
|                        |                                                |                          |

> Difference from ER model: in ER, *cardinality* meant mapping cardinality; here **cardinality = number of tuples**.

```css
  Customer Table — labelled:

  +-------------+--------+---------+---------+
  | Customer ID |  Name  | Address | Contact |
  +-------------+--------+---------+---------+   <-- These 4 columns = Degree of 4
  |      1      | Lakshya|   ...   |   ...   |   <-- Tuple 1  \
  |      2      |  Raj   |   ...   |   ...   |   <-- Tuple 2  /  Cardinality = 2
  +-------------+--------+---------+---------+

  Degree     = count of columns (attributes) = 4
  Cardinality = count of rows (tuples)        = 2
```
---
## 3. DB Design is a 2-Step Process
1. **Step 1:** Build the **ER Model** → gives the ER Diagram.
2. **Step 2:** Convert the ER Model into the **Relational Model** (tabular form).

> Any real-world problem / use case → design the **conceptual schema** in these two steps.
### Example: Online Delivery System
From an ER diagram, build tables:

**Customer table** (PK = Cust ID)
| Cust ID | Name | Address | Contact Number |
| ------- | ---- | ------- | -------------- |
**Order table** (PK = Order ID)
| Order ID | Time Stamp (when order placed) | Delivery Date |
| -------- | ------------------------------ | ------------- |
- **Each entity → a separate table (relation).**
- **Each attribute of that entity → a column** of the relation.
> A third step exists: implement the relational model in an **RDBMS software** (the software implementation of the relational model).>
> **RDBMS examples:** MySQL, Microsoft Access, Oracle.
---
## 4. Relational Model Definition & Properties
The relational model = data organized as a **collection of tables**. Each table has a **unique name** (e.g., Customer, Order).
- A **single row = one tuple** = one data point.
- **Columns = attributes.**
- **Relation Schema** — *defines the structure of the relation*: the relation name + its attributes. (e.g., the schema of Customer, the schema of Order.)
### Properties of a Table
1. **Name of the table is distinct** among all relations (every table has a unique name in the DB).
2. **Values must be atomic** — cannot be broken further (e.g., a contact number is atomic; a name is atomic).
3. **Name of each attribute is unique** within the relation.
4. **Each tuple must be unique** — no redundant/duplicate rows. (Don't store the same customer twice.)
5. **Sequence of rows and columns has no significance** — order of attributes/rows doesn't matter, as long as all attributes are defined.
6. **Tables must follow integrity constraints** — helps maintain data consistency across the table.
---
## 5. Relational Keys
A **relational key** = a set of attributes that can uniquely identify a tuple.
### Super Key
- **Any permutation/combination of attributes that can uniquely identify a tuple.**
- Examples (for Customer): `{Cust ID}`, `{Cust ID, Email}`, `{Name, Contact}`, etc.
- A super key **can contain redundant attributes** — i.e., it may include attributes beyond the minimal set needed for unique identification (a superset, in set-theory terms). A super key still uniquely identifies every tuple; the redundant attributes are just "extra."
### Candidate Key
- **The minimal subset of a super key** that can uniquely identify each tuple **and contains NO redundant attribute.**
- Two separate reasons disqualify an attribute or set from being a candidate key:
  1. **Not unique** — `Name` is disqualified because two customers can share a name; it is not a super key at all (it cannot uniquely identify every tuple).
  2. **Not minimal** — `{Cust_ID, Contact}` is disqualified because it contains a redundant attribute: either `Cust_ID` alone or `Contact` alone already uniquely identifies a tuple. A candidate key must be irreducible.
- Examples (minimal, unique keys): `{Cust ID}`, `{Contact}`.
- **A candidate key's attribute(s) cannot hold NULL values** — a NULL value cannot uniquely identify a tuple.
### Primary Key
- **One candidate key chosen by the database designer** to be the table's official unique identifier.
- Convention: choose the candidate key with the **fewest attributes** (simpler PKs are easier to index and use as foreign keys). Any candidate key is valid, but single-attribute keys are preferred.
- Example: both `{Cust ID}` and `{Contact}` are candidate keys; we choose `{Cust ID}` as the primary key.
### Alternate Key
- The **candidate keys NOT chosen as the primary key.**
- Formula: **Alternate Key = (set of Candidate Keys) − Primary Key**

### Prime & Non-Prime Attributes
- **Prime attribute** — an attribute that is part of **any candidate key** of the relation.
- **Non-prime attribute** — an attribute that is **not** part of any candidate key.

```c
  Example: Student_Project table
  Attributes: Student_ID, Project_ID, Student_Name, Grade
  Candidate key: {Student_ID, Project_ID}

  Prime attributes:     Student_ID, Project_ID  (part of the candidate key)
  Non-prime attributes: Student_Name, Grade      (not part of any candidate key)
```

> This distinction is central to normalization: 2NF and 3NF rules are defined in terms of what non-prime attributes depend on.

```c
  Key hierarchy — worked example:
  Table: Customer (Cust_ID, Name, Contact, Email)
  Assumption: Cust_ID is unique, Contact is unique, Name is NOT unique.

  SUPER KEYS (any combo that uniquely identifies a tuple — may have extras):
    {Cust_ID}
    {Contact}
    {Cust_ID, Name}       <-- redundant: Name is extra, Cust_ID alone works
    {Cust_ID, Contact}    <-- redundant: either alone would work
    {Cust_ID, Email}      <-- redundant
    {Cust_ID, Name, Contact, Email}  ... and many more

  CANDIDATE KEYS (minimal — no attribute can be removed and still uniquely identify):
    {Cust_ID}   <-- minimal; can't remove anything
    {Contact}   <-- minimal; can't remove anything
    (NOT {Cust_ID, Contact}: removing either Cust_ID or Contact still gives uniqueness
     -> not minimal -> this is a super key, not a candidate key)

  PRIMARY KEY (chosen candidate key; pick the one with fewest attributes):
    {Cust_ID}   <-- chosen as PK

  ALTERNATE KEY (candidate keys not chosen):
    {Contact}

  Relationship:  All Candidate Keys ⊆ All Super Keys
                 PK + Alternate Keys = All Candidate Keys
```

---

## 6. Foreign Key (Most Tricky & Important)

### Why it exists
- Two entities (Customer, Order) with a relationship (`Places`). After converting both to tables, *how do we represent the `Places` relationship in tabular form?* → use a **Foreign Key**.

### How it works
- Take the **Primary Key of one relation** and add it as an attribute inside the **other relation** → that added attribute is the **Foreign Key**.

**Example:** Add `Cust ID` into the **Order** table.

**Order table**

| Order ID | Time Stamp \| Delivery Date | Cust ID (FK) |
| -------- | --------------------------- | ------------ |
| 21       | ....                        | 1            |
| 22       | ....                        | 2            |
| 23       | ....                        | 3            |

- `Cust ID` is the **PK of Customer** but appears as a **FK in Order**.
- This establishes the cross-reference: to find which customer placed order 21 → look at its `Cust ID = 1` → go to Customer table → retrieve full info for customer 1.

```c
  How the FK cross-reference works:

  Customer Table (Parent / Referenced)     Order Table (Child / Referencing)
  +----------+--------+                    +----------+------+---------+
  | Cust ID  |  Name  |                    | Order ID | ...  | Cust ID |
  +----------+--------+                    +----------+------+---------+
  |    1     | Lakshya|<--+                |    21    | ...  |    1    |--+
  |    2     |  Raj   |   |                |    22    | ...  |    2    |  |
  +----------+--------+   |                |    23    | ...  |    3    |  |
                          +-----------------------------------------------+
                          Order 21's Cust_ID=1 points to Customer row 1.
                          This is how the "Places" relationship is stored in tables.

  The PK in Customer (Cust_ID) becomes the FK in Order (Cust_ID).
  Same column name, but it plays a different role in each table.
```

### Formal definition
> A relation `r1` may include among its attributes the **PK of another relation `r2`** — that attribute is the **foreign key** of `r1` referencing `r2`.

### Parent / Child terminology
| Table where FK is added | Table whose PK is used |
|-------------------------|------------------------|
| **Child Table** / **Referencing Relation** (Order) | **Parent Table** / **Referenced Relation** (Customer) |

> The foreign key provides a *cross-reference* between two relations (establishes a parent–child relationship).

---

## 7. Composite, Compound & Surrogate Keys

| Key               | Definition                                                                                                                                                     |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Composite Key** | A primary key formed using **at least two attributes**.                                                                                                        |
| **Compound Key**  | A primary key formed using **two foreign keys**.                                                                                                               |
| **Surrogate Key** | A system-generated synthetic key (an auto-generated **integer** value) assigned to each tuple. Auto-increments (1, 2, 3, ...). Can be used as the primary key. |

### Surrogate key example (school merge)
- School A registration numbers: 101, 102, 103...
- School B registration numbers: AB101, AB102...
- When **merging** the two school tables, the registration number **cannot be the PK** (formats differ / collide).
- Solution: ask the DB to assign a **self-generated integer** (surrogate key) to each tuple, then merge. This synthetic integer PK uniquely identifies each row in the merged table.

```c
  Why a surrogate key is needed for the school merge:

  School A rows:         School B rows:
  Reg_No | Name          Reg_No | Name
  101    | Avi           AB101  | Bob
  102    | Sai           AB102  | Cat

  Problem: if we merge into one table and use Reg_No as PK,
  "101" (integer) and "AB101" (string) are different types,
  and the numbering systems overlap in meaning (both call something "1").
  We cannot use Reg_No as a single consistent PK across both schools.

  Fix — let the DB assign its own integer (surrogate key):

  Merged Table:
  Sys_ID | Reg_No | Name        Sys_ID is auto-generated (1, 2, 3, 4...)
  1      | 101    | Avi         It is the new PK.
  2      | 102    | Sai         Reg_No is kept as data but is NOT the PK.
  3      | AB101  | Bob
  4      | AB102  | Cat
```

---

## 8. CRUD Operations

**Create, Read, Update, Delete** — the operations performed on a table:

| Operation  | Meaning                                                      |
| ---------- | ------------------------------------------------------------ |
| **Create** | Add a new entry/tuple                                        |
| **Read**   | Read data (e.g., look up order 21; nothing modified)         |
| **Update** | Modify an existing entry (e.g., change a customer's address) |
| **Delete** | Remove an entry                                              |

> On every CRUD operation, the DB applies **integrity policies / consistency checks**. If they're violated, the DB becomes **inconsistent/corrupt**, so the operation must be rejected.
---
## 9. Integrity Constraints
### How inconsistent data arises (example)
Inserting a Student with `Roll No = z` (a string), `Name = 1812` (a number), `Contact = ABC` → this is **not consistent**. To prevent it, the DBMS provides **integrity constraints**.
### (a) Domain Constraint
- Restricts the **domain** (allowed data type / value range) of an attribute.
- Example: `Roll No` domain = integer; `Name` = characters; `Contact` = integer.
- Can also restrict value ranges: e.g., `Age >= 18` for college admission, or DOB earlier than 2002 for a job.
- On a CRUD operation (e.g., inserting `z` into an integer `Roll No`), the DB **throws an error** and the operation fails.
### (b) Entity Constraint
- **Every table must have a Primary Key**, and that **Primary Key cannot be NULL** (`PK ≠ NULL`). Otherwise you can't uniquely identify rows.
### (c) Referential Constraint (very important for interviews)
Constraints between **parent (referenced)** and **child (referencing)** tables, using the foreign key.
*Insert Constraint*
> A value **cannot be inserted into the child table** if the value is **not lying in the parent table**.
- Example: cannot add an Order with `Cust ID = 4` if customer 4 does not exist in Customer. The DBMS throws a **foreign-key constraint violation** error.
*Delete Constraint*
> A value **cannot be deleted from the parent table** if that value is **lying in the child table**.
- Example: cannot delete `Cust ID = 3` from Customer while order 23 (`Cust ID = 3`) still references it — otherwise the FK reference would dangle, creating inconsistency.
```c
  Referential Constraint — both rules illustrated:

  Customer (Parent):  Cust_ID = {1, 2, 3}
  Order (Child):      contains Cust_ID 1, 2, 3 as FK values

  INSERT rule:
    Try to insert Order with Cust_ID = 4
    --> DB checks: does 4 exist in Customer.Cust_ID? NO.
    --> REJECTED. (Can't reference a customer that doesn't exist.)

  DELETE rule:
    Try to delete Customer where Cust_ID = 3
    --> DB checks: does 3 appear in Order.Cust_ID? YES (Order 23).
    --> REJECTED. (Deleting would leave Order 23 pointing at nothing.)

  Both rules protect the FK link from becoming a "dangling pointer".
```
---
## 10. Handling Delete — ON DELETE Options (Interview Favorite)
How to delete from the parent even when the child references it, **without violating** the delete constraint:
### ON DELETE CASCADE
- When deleting a value from the **parent** table, **also delete the corresponding entry from the child** table.
- Deleting `Cust ID = 3` from Customer → its corresponding Order entry is also deleted. Both gone → no inconsistency → delete constraint satisfied.

**SQL (concept-level, syntax taught later):**
```sql
CREATE TABLE Order (
    ...
    Cust_ID INT REFERENCES Customer ON DELETE CASCADE
    ...
);
```

### ON DELETE SET NULL (Can a Foreign Key be NULL? — Yes!)
- Instead of deleting the child entry, **set the foreign key value to NULL** in the child.
- Deleting `Cust ID = 3` from Customer → the order's `Cust ID` becomes **NULL** (an "orphan" order, not pointing to any customer). Delete constraint not violated.
- The correct SQL clause is **`ON DELETE SET NULL`** (not "ON DELETE NULL" — that is not valid SQL syntax).

```c
  ON DELETE CASCADE vs ON DELETE SET NULL:

  Before:
    Customer: [Cust_ID=3, Name=Raj]
    Order:    [Order_ID=23, Cust_ID=3]

  After DELETE Customer Cust_ID=3 with CASCADE:
    Customer: [row 3 gone]
    Order:    [Order_ID=23 ALSO gone]    <-- child row deleted too

  After DELETE Customer Cust_ID=3 with SET NULL:
    Customer: [row 3 gone]
    Order:    [Order_ID=23, Cust_ID=NULL] <-- FK set to NULL, order kept

  CASCADE: use when the child record has no meaning without the parent.
  SET NULL: use when the child record can exist independently
            (order history kept even if customer account deleted).
```

> **Interview Q: Can a foreign key have a NULL value?** → **Yes.** A FK becomes NULL when the parent's corresponding PK row is deleted (the referenced entity has left the system) — handled via `ON DELETE SET NULL`.

---

## 11. Key Constraints (defined at `CREATE TABLE` time)

Constraints applied to a particular attribute when defining the relation. The constraint must be followed on every CRUD operation, else the operation fails.

| # | Constraint | Meaning |
|---|-----------|---------|
| 1 | **NOT NULL** | Column must not accept NULL values. By default an attribute *can* be NULL; `NOT NULL` enforces otherwise. |
| 2 | **UNIQUE** | The attribute's values stay **unique across the whole table**. Similar to PK, but **you may have many UNIQUE constraints per table** (vs. only one PK). |
| 3 | **DEFAULT** | Sets the **default value** of a column (e.g., Prime status default). |
| 4 | **CHECK** | Limits the value range/domain (e.g., `CHECK (Age >= 18)`). On insert with `Age = 17`, the DB throws a consistency error. |
| 5 | **PRIMARY KEY** | Only **one** PK per relation; uniquely identifies tuples. |
| 6 | **FOREIGN KEY** | Keeps the relationship between two tables. |

**Example syntax (concept-level):**
```sql
CREATE TABLE Customer (
    ID      INT          NOT NULL,
    Name    VARCHAR(50)  NOT NULL,   -- VARCHAR(50) = up to 50 characters
    ...
    ID      ...          UNIQUE,
    ...
    ID      INT          PRIMARY KEY
);
```

**Foreign-key constraint syntax (Order example):**
```sql
CREATE TABLE Order (
    ...
    PRIMARY KEY (Order_ID),
    FOREIGN KEY (Cust_ID) REFERENCES Customer(Cust_ID)
);
```

> **Foreign Key Constraint summary:** Whenever there is a relationship between two entities, there must be a **common attribute** between them. This common attribute is the **PK of one entity set** and **becomes the FK of the other.**

---

## 12. Summary

- Relational model = data organized in **tables (relations)**; implemented via **RDBMS** software (Oracle, IBM, MySQL, MS SQL).
- DB design = ER model → Relational model → implement in RDBMS.
- **Keys:** Super → Candidate → Primary → Alternate; plus Foreign, Composite, Compound, Surrogate.
- **Integrity constraints:** Domain, Entity, Referential (Insert/Delete constraints, `ON DELETE CASCADE` / `ON DELETE SET NULL`).
- **Key constraints:** NOT NULL, UNIQUE, DEFAULT, CHECK, PRIMARY KEY, FOREIGN KEY.

**Next lecture:** Converting an ER Model into a Relational Model (with examples).
