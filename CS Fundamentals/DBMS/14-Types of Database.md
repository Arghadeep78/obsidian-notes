## 1. Relational Databases (RDBMS)

### Concept & Logic
- Based on the **Relational Model.** Data is defined in the form of **tables**; tables (also called **relations**) are **related to each other** via **foreign keys** → **referential integrity**, with many **constraints**.
- Meaningful data is retrieved using **JOINs** (establishing relations across tables) via **SQL**.

> **What a FK relation means in practice:** The `dept_id` column in the Student table does not store the full department name — it stores a reference (key) to the Dept table. The actual department name lives in the Dept table. To get "Alice's department name", you must JOIN both tables at query time. This is the core trade-off of normalization: no redundancy, but joins are required to reassemble meaning.

```css
RELATIONAL MODEL — EXAMPLE:

  Student Table:              Dept Table:
  ┌──────┬───────┬────────┐   ┌────────┬──────────┐
  │ s_id │ name  │ dept_id│   │dept_id │ dept_name│
  │  1   │ Alice │   10   │──►│  10    │ CS       │
  │  2   │ Bob   │   20   │──►│  20    │ Math     │
  └──────┴───────┴────────┘   └────────┴──────────┘
          FK relationship: Student.dept_id → Dept.dept_id

  Query: "Get Alice's department name"
  → JOIN Student + Dept ON dept_id = dept_id → "CS"
```

### Properties
- **Most popular and most mature** DBMS; very old → **excellent community support** (Stack Overflow, courses, easy help when stuck).
- Stores **structured data** in **discrete tables**; **highly optimized.**
- Provides a stronger guarantee that data is **normalized** (you design ER diagram → convert to tables → apply normalization → remove redundancy) → **no data redundancy.**
- Usable in **almost all use cases** *except* where **horizontal scalability** is required (the cloud use case from the NoSQL lecture).

### Disadvantages
- **Huge systems become complex** — tracking which data relates to what gets messy → harder for the **DBA** to manage when the relational model becomes very large.
- **Biggest disadvantage = Scalability issue** — **horizontal scalability** is the main problem.
  > Why the relational model isn't used in cloud systems / isn't the "best" DBMS: it **cannot fulfill today's cloud (horizontal scaling) requirements** well.

### Examples
- **MySQL**, **Microsoft SQL Server**, **Oracle**.

---

## 2. Object-Oriented Databases (OODB)

### Concept & Logic
- Based on the **Object-Oriented Programming (OOPs) paradigm** — works on concepts similar to OOPs (classes, objects, encapsulation, inheritance, polymorphism).
- **Class** = blueprint with **attributes** (characteristics / data types) and **methods** (define the object's **behavior** — what the object can do).
- In OODB, **everything is defined and stored as objects.** Data is **not stored directly** as string/data-type; you **encapsulate/bundle** data into a **class's object** and store that **object directly** on disk / in the DB. On retrieval, data comes back **as an object.**
- Objects can contain tables / **executable code** (in their methods). **Objects communicate via methods.**
- **Handles** = the object's id (≈ primary key in relational terms).
- "All information stored in the DB is capable of being represented as an object."

> **Why this matters:** In a relational DB, a `Student` row is just raw data — (1, "Alice", 21). To use it in an OOP application, you'd manually map each column to a field. In OODB, the `Student` object is stored directly: its attributes AND its methods (e.g., `isPassed()`, `averageMarks()`) are part of the stored entity. The DB and the application speak the same language — objects.

### Key OOP concepts applied
- **Inheritance:** e.g., a `Person` class (name, gender, age); `Student` and `Professor` **inherit** from `Person` (each adds its own attributes). `Student is a Person`, `Professor is a Person`.
- **Encapsulation:** Hide which attributes of an object are visible to whom (and vice versa). Methods provide an **interface** to the object.
- **Object Identity:** The object itself represents a data point (a `Student` object represents that it's a student).
- Stores **structured data** (needs a structure to bundle into the class/object).

```css
OODB — CLASS HIERARCHY & OBJECT INTERACTION:

             ┌──────────────────────────┐  
             │  Person (superclass)     │
             │  - name, phone, email    │------------------------
             │  - address (Address obj) │                       | 
             │  + livesAt()             │
             └───────────┬──────────────┘                       |
                         │ inherits 
          ┌───ofType()───┴──────────────┐                       |  
          │                             │
  ┌───────┴──────────┐                 ┌──────────┴──────-──┐
  │ Student          │                 │ Professor          │   |
  │ - roll_no, marks │                 │ - faculty_id       │
  │ + isPassed()     │◄──----------────│ can call           │   |
  │   (checks        │ isPassed access │ student.isPassed() │
  │    past records) │      given      └──────────────────-─┘   |
  └──────────────────┘                                       looksAt()

             ┌──────────────────────────┐                       |
             │  Address                 |------------------------
             │  - street, city,...      │
             │  - address (Address obj) │
             │  + livesAt()             │
             └───────────┬──────────────┘

  - Data = objects stored directly on disk
  - Objects talk via methods (isPassed, livesAt)
  - averageMarks can be hidden via encapsulation (not shown to Professor)
```

### Example walkthrough
- `Person` (name, phone, email, address) → `Student` (roll_no, marks…) and `Professor` inherit it.
- `Student.isPassed()` method (exposed to Professor) checks a related past-records DB whether the student passed in 2021, how many subjects failed, etc.
- `Person.livesAt()` method returns the address (stored as a separate Address object).
- `Student.averageMarks` can be **hidden** (not exposed) — encapsulation.
- → Objects interact with each other **through methods.**

### When to use
- When the database becomes **very complex** and maintaining **relations is tedious** in the relational model (relational model's basis is relations; if establishing/maintaining them is hard, shift to OODB).
- All needed information comes in **one instantly-available object package** instead of multiple tables.

### Advantages
- **Data storage & retrieval is easy and quick** — data comes as an object; most info available instantly → **no joins** needed. *(Somewhat NoSQL-like, but not entirely.)*
- Can **handle complex data + more variety of data types** than relational DBs (no strict "everything must be an individual table / heavy normalization" rule).
- **Relatively friendly to model the real world** — OOP exists to model real-world problems easily.
- **Works well with OOP/object languages** — if your software (C++/Java, object-oriented) and your DB are both object-oriented, their interaction is easy (same paradigm).

### Disadvantages
- **High cost / performance issues** (e.g., on writes) — high complexity from calling methods.
- **Low community support** (vs mature relational DBs).
- **Views not supported** like relational DBs (in relational you declare a view → a trimmed-down version of student visible to a professor). In OODB you can partially mimic this through **method visibility and access modifiers** (e.g., making `averageMarks` a private method that a Professor object cannot call), but this is not the same as a declarative relational view — it requires coding access rules into the class methods themselves.

### Examples
- **ObjectDB**, **GemStone**.

---

## 3. NoSQL Databases (quick review)

- **Non-tabular** databases; data stored in ways **other than relational tables.** **Schema-free** (no tables).
- Can handle **huge data**; has **redundancy / repeated data.**
- *(Full details in file 13.)* Prime example: **MongoDB.**

---

## 4. Hierarchical Databases

### Concept & Logic
- Information is stored in a **hierarchy** (tree-like, **inverted tree**). There is a **single root**, then **parent → child** relationships (**one-to-many**). Each child has **exactly one parent.**

> **Why "inverted tree":** The root sits at the top and branches downward — like a family tree drawn with the oldest ancestor at the top. The file system on your computer works exactly this way: one root directory, branching into child directories, each with exactly one parent. The source explicitly gives the file system as an example of hierarchical DB use.

### Examples / Use cases
- **Family tree** (grandparents → parents → individuals → their children).
- **File system** (movies, music → directories → subdirectories) — parent-child structure.
- **Organization management** — CEO → VPs/President → CTO → Senior Directors → Directors → Managers (marketing/HR/CA departments); store who reports to whom, which department under which.
- **Use when** information must be retrieved **top-to-bottom** via traversal (e.g., "list all my TVs" in an electronics shop: Electronics → Phones/TVs/Washing-Machines → TV → LED/LCD nodes).

```css
HIERARCHICAL DATABASE — ELECTRONICS SHOP EXAMPLE:

                    [Electronics]         ← ROOT
                   /      |       \
               [Phones] [TVs] [Washing Machines]
                          |
                    ┌─────┴──────┐
                   [LED]        [LCD]

  Rules:
  - Each node has EXACTLY ONE parent (TVs → Electronics only)
  - Parent can have multiple children (one-to-many)
  - Traversal is always top → bottom (start from root)

  Query "list all TVs": Root → Electronics → TVs → list [LED, LCD] ✅
  Easy and fast because structure mirrors the real-world hierarchy!
```

### Schema & Physical model
- Schema defined by a **tree-like (inverted-tree) organization**: a root/parent directory of data stored as a **record** linking to various **subdirectories**, each branching to child records.
- **Big advantage:** Both the **logical schema** (conceptual) and the **file system (physical schema)** are **tree-like**, so the logical schema can be **directly copied/derived** into the disk/physical model. *(Recall conceptual vs physical vs view schema.)* → can be used in the **physical model** too (disk storage system is also hierarchical).

### Advantages
- **Easy to use** — simple structure (e.g., electronics shop).
- **One-to-many organization** makes **traversal simple and fast** — ideal for **website drop-down menus**, **computer folder systems**.
- **Separation of table from physical storage** → information can be **easily updated/deleted without affecting the rest** of the DB. *(Drop the "TV" node → all its children auto-delete → DB freed easily.)*
- **Most major programming languages** support tree functionality (trees are fundamental in DSA).

### Disadvantages
- **Inflexible** by nature. The **one-to-many** structure **cannot describe relationships among siblings** (same level) or where a **child has multiple parents.** Not ideal for complex relationships.
- **Tree-like organization requires top-to-bottom sequential search** → **time-consuming** for very large trees (must always start search from the **root**) → costly traversal.
- Requires **repetitive storage of data** in multiple different entities, which can be **redundant.**

```css
HIERARCHICAL — LIMITATION EXAMPLE:

  Student is in CSE dept AND in Chess Club.
  In hierarchical DB, a node can only have ONE parent:

  ┌────────┐           ┌──────────┐
  │  CSE   │           │Chess Club│
  └────────┘           └──────────┘
       │                    │
  [Student?]           [Student?]     ← Can't have 2 parents!
       ↑
  Must pick ONE parent → cannot model this relationship ❌
  → Use Network DB instead.
```

### Example
- **IBM IMS** (Information Management System).

---

## 5. Network Databases

### Concept & Logic
- The **fifth type** — basically an **extension** of the hierarchical database. Still hierarchy-like, **but a node can have MULTIPLE parents.**
- Because of multiple parents → it's a **graph-like structure** (graph organization). Child records have the **freedom to associate with multiple parent records.**
- → **Handles complex relationships** that hierarchical DBs cannot.

```css
NETWORK DATABASE — MULTIPLE PARENTS (graph structure):

  The key difference from hierarchical: a child can have MORE THAN ONE parent.

  University (root)
       │
       ├──────────────────────────┐
       │                          │
  [Departments]               [Clubs]
    │                           │
  [CSE][Math][Physics]    [Chess][Drama][Sports]
    │                           │
    └──────────┐   ┌────────────┘
               │   │
           [Student]  ← has TWO parents: Departments AND Clubs
           [Faculty]  ← has TWO parents: Departments AND Clubs

  In a hierarchical DB, [Student] can only sit under ONE parent — impossible here.
  In a network DB, the graph structure allows [Student] to belong to both.

  Hierarchical DB → ❌ (only 1 parent allowed per node)
  Network DB → ✅ (multiple parents = graph, handles m:n relationships)
```

### Example
- **University** (root) → Departments, Admin → Students, Faculty, Sports → Clubs.
- A **Student's parents** = Department **and** Clubs; **Faculty's parents** = Department **and** Clubs. → **Multiple parents** → use a Network DB.

### Disadvantages
- Supports **many-to-many (m:n) links** → traversal becomes even **slower** than hierarchical (graph traversal is costly), especially with large data.
- **Poor web/community support; not very famous** → very **specific use cases**, rarely used (past or future).
- Maintainance is tedious: as changes to data structure require updating many explicit pointer-based relationships between records.

### Examples
- **IDS** (Integrated Data Store), **IDMS** (Integrated Database Management System), **Raima Database Manager**, **TurboIMAGE** — old **legacy** systems.

---

## 6. Summary & Practical Guidance

| Type | Model / Structure | Key trait | Use case | Examples |
|---|---|---|---|---|
| **Relational (RDBMS)** | Tables + FK relations | Mature, normalized, structured; **scalability issue** | General-purpose (non-cloud) | MySQL, MS SQL Server, Oracle |
| **Object-Oriented** | Objects (OOP) | Complex data, OOP-friendly; low community support | Complex relations, OOP software | ObjectDB, GemStone |
| **NoSQL** | Non-tabular, schema-free | Horizontal scaling, huge data, redundancy | Cloud apps | MongoDB |
| **Hierarchical** | Tree (one-to-many, single parent) | Easy traversal top→bottom; inflexible | Folder systems, dropdown menus, org charts | IBM IMS |
| **Network** | Graph (multiple parents, m:n) | Handles complex relations; slow, rare | Multiple-parent structures | IDS, IDMS, TurboIMAGE |

```css
ALL 5 TYPES — QUICK VISUAL REFERENCE:

  RELATIONAL:     OBJECT-ORIENTED:    NoSQL:
  ┌───┬───┐       [Person obj]        { "name": "Alice",
  │row│row│       [Student obj]         "hobbies": [...] }
  │row│row│       [Prof obj]
  └───┴───┘       ↑ OOP paradigm      Self-contained documents

  HIERARCHICAL:   NETWORK:
      (root)          (root)
     /    \          /    \
  [child] [child]  [A]    [B]
     |                \  /
  [grandchild]        [C]    ← C has 2 parents (A and B)
  One parent only.    Multiple parents allowed.
```

### Most Important (focus areas)
- **Most famous / market-used:** **Relational Databases** and **NoSQL Databases.**
- **NoSQL** = modern databases (heavily used in **cloud** companies). **Relational** = old, very mature.
- Hierarchical, Network, and Object-Oriented DBs have **very specific use cases** and **little community support** → not widely used.

### How to choose (decision hints)
- **Relational** → general structured data, no horizontal-scaling need.
- **NoSQL** → cloud / horizontal scaling / unstructured huge data.
- **Hierarchical** → data is naturally a tree, no sibling relations, single parent.
- **Network** → nodes need **multiple parents** (graph-like).
- **Object-Oriented** → OOP software + complex object relations.

> The key types to know: relational and NoSQL in detail, then the other types (object-oriented, hierarchical, network) with their use cases, advantages/disadvantages, and community/developer support.
