## 1. Abstraction

**Concept & Logic:**

- **Abstraction = simplifying the system for users** by exposing only what the user needs to know and **hiding the underlying complexity** (how things work internally).
- _System hides certain details_ → interaction becomes easy and simplified for the user.

**Examples:**

- **Car driving:** you only handle accelerator, brake, clutch, steering. You don't need to know how the axle rotates or how force is balanced on tyres. That's the engineers' job. → that's abstraction.
- **Tally / Excel (business software):** built by developers, used by commerce users. The user clicks to add/delete/calculate (e.g., GST), exports a balance sheet to PDF — without knowing the internal data structures, encoding, encryption, or compression used to save the file.

```
Abstraction = "what you see" vs "what's hidden"

  Car user sees:              Hidden from user:
  ┌───────────────┐           ┌─────────────────────────┐
  │  Accelerator  │           │  Engine combustion      │
  │  Brake        │  ──────►  │  Axle force balance     │
  │  Steering     │           │  Differential gearing   │
  └───────────────┘           └─────────────────────────┘
     (simple interface)              (complex internals)
```

**Abstraction in DBMS:** A major purpose of DBMS is to provide an **abstract view of data** to the user. How data is stored/maintained, which data structures are used (hash, table, bits/bytes) — the user doesn't care; the user just needs to _see_ the data.

---

## 2. View of Data — Purpose of Abstraction (Amazon example)

Multiple users access the **same database**, but each needs a **personalized view**.

**Amazon DB stores (per customer):** name, phone, address, age, liking/disliking, most-used credit card, debit card, UPI address, customer reviews, products bought, etc.

- **Logistics Department** needs only **Name, Phone Number, Address** (to deliver — send trucks/delivery boys). It has no business knowing credit card / UPI / payment details.
- **Customer Service Department** needs **Name, Phone, Address + Products previously ordered** (for refund/replace, to know the last order).

```
           ┌─────────────────────────────────────────┐
           │           Amazon Database               │
           │  Name | Phone | Address | Credit Card   │
           │  UPI  | DOB   | Orders  | Reviews  ...  │
           └─────────────────────────────────────────┘
                   │                      │
           ┌───────▼──────┐      ┌────────▼──────────┐
           │  Logistics   │      │  Customer Service  │
           │  View        │      │  View              │
           │──────────────│      │────────────────────│
           │ Name         │      │ Name               │
           │ Phone        │      │ Phone              │
           │ Address      │      │ Address            │
           └──────────────┘      │ Products Ordered   │
                                 └────────────────────┘
        Credit card / UPI hidden from both views
```

> **Goal of abstraction in DBMS:** multiple users access the same data, but each gets a personalized view — irrelevant details are hidden (beautifully hidden using abstraction).

---

## 3. Three-Schema Architecture (How Abstraction is Achieved)

> **Schema = Design.** (Replace the word "schema" with "design" — how a thing looks / how it is built.)

The DBMS is divided into **three abstract levels** to provide abstraction:

```
  ┌──────────────────────────────────────────┐
  │         VIEW LEVEL (External)            │  ← end users see this
  │   View 1 (Logistics) | View 2 (CS) ...   │
  └──────────────────────┬───────────────────┘
                         │ hides physical details
  ┌──────────────────────▼───────────────────┐
  │       LOGICAL LEVEL (Conceptual)         │  ← DBA works here
  │  Tables, relationships, constraints      │
  └──────────────────────┬───────────────────┘
                         │ hides storage details
  ┌──────────────────────▼───────────────────┐
  │       PHYSICAL LEVEL (Internal)          │  ← how data sits on disk
  │  B-trees, compression, block sizes ...   │
  └──────────────────────────────────────────┘
```

### 3.1 Physical Level (Internal Level) — Lowest level of abstraction

- Describes **how data is actually stored** on disk/SSD.
- Example: a profile picture stored in the DB — at the physical level you describe how it is actually stored (e.g., compress the image with **Run-Length Encoding** before storing: 1 MB → 50 KB).
- Defined by the **Physical Schema** — a **blueprint** (like an architect's house map: bedroom here, gallery there, stairs here) that specifies: which compression, which encryption, where to store, block sizes (large/small), whether to hash, which data structures.
- **Goal:** store data so that **access is fast**. (E.g., don't use a compression whose decompression is too CPU-costly.) May use Binary Tree / Red-Black Tree / n-ary Tree / Tries to store data for fast search.

### 3.2 Logical Level (Conceptual Level) — Middle level

- Describes **what data is stored** and the **relationships** among the data — _not_ how it's physically stored.
- **Example (Student):** physically, a file just has comma-separated values (`Ram, 88123, w526, batch 201, ...`). Looking at the raw file you cannot tell it's "Student" data. You **transform (map)** the physical file → logical level to get a table: `Name | Phone Number | Address | Batch Number`. Now you can logically say _"this is Student data with these attributes."_
- This **physical-to-logical mapping** is done by the **Logical Schema (Conceptual Schema)**.

```
  Physical file (raw):         Logical view (after mapping):
  ─────────────────────        ──────────────────────────────────────
  Ram,88123,w526,batch201  →   Name | Phone    | Address | Batch
  ...                          Ram  | 88123     | w526    | batch201
```

- **Relationships:** e.g., a `Course` table (`Course ID, Name, Duration`) — every student enrolls in 5 courses, so Student has a `Course ID`. The logical level describes the **relationship**: _"Student applies for a Course / Courses are applied by Students."_ (At the physical level both `Name`s look like strings and `Phone`/`Course ID` look like integers — you can't tell them apart.)

### 3.3 View Level (External Level) — Highest level of abstraction

- Used by the **end user**. The user here doesn't care about conceptual or physical/internal details.
- Defined by the **View Schema**: "describes the database part that a particular user group is interested in, hiding the rest."
- **Example:** Logistics view schema = `Name, Address, Phone`. Customer Service view schema = `Name, Address, Phone, Products Bought`. The DB is the same (customer data stored once), but **sub-schemas** differ → different views.
- At the external level a DB contains several schemas, sometimes called **sub-schemas**.
- **Security mechanism** is also applied at this level: some info (security codes, OTP, secret codes) shouldn't be shown to certain groups (e.g., logistics).

> **Main objective of 3-level architecture:** enable multiple users to access the **same data** with a **personalized view**, while storing the **underlying data once** (DB stored in one place).

---

## 4. Physical Data Independence

**Concept & Logic:**

- **Physical Data Independence = if you change the physical schema, the logical level / application is NOT affected.**
- Example: move the file from disk → **SSD** (for faster access). The user at the logical level is unaffected — only a **new physical-to-logical mapping** is defined.
- Example: split a single file (`Name, Phone, Address, Batch`) into **four separate files** (all names in one, all phones in another, etc.). The mapping changes (1st string of file 1 = name, 1st integer of file 2 = phone, …), but **Student still looks the same** at the logical level (still has Name, Phone, Address, Batch).
- The logical level only talks about **which data is stored** and **its relationships**, never _how_ it is stored.

```
  Physical change (e.g., HDD → SSD, or 1 file → 4 files):

  Before:                         After:
  [single file on HDD]      →     [4 files on SSD]
        │                                │
  Physical-to-logical mapping      New mapping defined
        │                                │
  Logical level: Student (Name, Phone, Address, Batch)
        │                       (UNCHANGED)
  Application: Student class with same fields
        │                       (UNCHANGED)
  → Physical Data Independence achieved
```

---

## 5. Instance vs Schema

### Instance

- **Instance = the collection of information stored in the DB at a particular moment.**
- Example: Student DB just created → 1st student enrolls, 2nd enrolls. At 12 PM the instance = 2 rows (2 data points: student XYZ, student ABC). Next day it might be 3 students; after a year, thousands. If a student withdraws, the instance reflects 2 students again. The **table stays the same**; data inside keeps filling/changing.

```
  Schema (structure, rarely changes):
  Student: [ID | Name | Phone | Address | Batch]
                     ↑ fixed blueprint

  Instance at 12 PM (snapshot of data right now):
  ┌────┬──────┬───────────┬──────────┬────────┐
  │ ID │ Name │   Phone   │ Address  │ Batch  │
  ├────┼──────┼───────────┼──────────┼────────┤
  │  1 │ XYZ  │ 99XXXXXXX │ Delhi    │ 2024   │
  │  2 │ ABC  │ 88XXXXXXX │ Mumbai   │ 2024   │
  └────┴──────┴───────────┴──────────┴────────┘
  Instance changes over time; schema stays the same.
```

### Schema

- **Schema = the overall design.** Three types: Physical-level schema, Logical-level schema, View-level schema (view schema is multiple → sub-schemas).
- **Nomenclature note:** the normally-used term **"DB Schema" = Logical-level Schema** (also called Conceptual Schema). Schema describes how structural data is described (Student's design = name, address, batch).
- Schema **does not change often**; physical-level things may change, but Student still "looks" the same.

**What a DB (Logical) Schema contains:**

1. **Attributes** of the table (e.g., Student's attributes).
2. **Consistency Constraints** — e.g., Name cannot be null (no student without a name); also add a **Student ID** for **unique identification** (Primary Key cannot be null).
3. **Relationships** — e.g., _Student applies for a Course._

> **Logical schema is the most important** in terms of its effect on applications — the programmer constructs apps using the logical schema. (When you build an app's `Student` class, its members — name, address, phone, student ID — mirror the logical schema. If you forget `name` in your app's structure, that's wrong.)

---

## 6. Data Models

**Concept & Logic:**

- A **data model** provides **a way to describe the design of the DB at the logical level** — the **underlying structure of the DB**.
- It describes: how the whole data is structured and how different tables relate (e.g., banking: how Employees relate to Payroll, how a Customer relates to Debit/Credit card info, how a Transaction relates to a Customer — Transaction table separate, Customer table separate, how they relate).

**A data model = a collection of conceptual tools for describing:**

1. **Data** (e.g., a Student is described by 5 things).
2. **Data Relationships** (relation between Course and Student).
3. **Data Semantics** (meaning of the data).
4. **Consistency Constraints** (e.g., bank account balance shouldn't go below 0).

**Types of Data Models (named in the video):**

- **Relational Model**
- **Entity-Relationship (ER) Model**
- **Object-Oriented Model**
- **Object-Relational Model** (= Object-Oriented Model **mixed with** Relational Model)

> ER Model & Relational Model are covered in detail in the coming classes. A data model is used to **logically define/describe** data.

---

## 7. Database Languages

To interact with the DBMS you need a language (just like programs are written in a language the compiler understands → machine code). Two kinds of statements:

### 7.1 DDL — Data Definition Language

- **To specify the database schema** (define the schema first, at the logical level).
- Also: while defining the schema with DDL, **we specify consistency constraints** (e.g., Student ID not null, Name not null, Phone must be 10 digits). These are **checked on every insert/delete/update/modification** — a student entering a 5-digit mobile number will get an **error** and won't be inserted.
- **Example (SQL):**
    
    ```sql
    CREATE TABLE Student (
        ST_ID    INT,
        Name     VARCHAR(50),   -- consistency constraint: name ≤ 50 characters
        Address  VARCHAR(...),
        Phone    INT
    );
    ```
    

### 7.2 DML — Data Manipulation Language

- **To manipulate data:** retrieve, insert, delete, update.
- Retrieval example: Principal asks "list all students living in a particular PIN code/address" → you need a data-retrieval mechanism.
- **Queries** are part of DML — statements requesting the retrieval of information.
- **Example (SQL):**
    
    ```sql
    SELECT * FROM Student;   -- * (asterisk) = everything; lists all students
    ```
    

### 7.3 SQL — one language, both DDL & DML

- **DDL and DML are NOT two separate languages** (it's not like C++ vs Java vs Python). Both features are provided in a **single DB language**.
- **SQL (Structured Query Language)** provides **both**: DDL mechanisms (define/modify schema) and DML mechanisms (manipulate/retrieve data).
- **How a query runs:** you write a query → it's sent to the DBMS/DB → the DBMS runs it, understands it, fetches information from the DB, and returns the result.

```
  DDL (define structure):          DML (work with data):
  CREATE TABLE, ALTER TABLE        SELECT, INSERT, UPDATE, DELETE
  DROP TABLE, CREATE INDEX         ...
          └──────────────┬──────────────────┘
                         │
                   Both are SQL
```

---

## 8. Accessing the DB from an Application (Host Language Connectivity)

**The problem:** Your application is written in a **host language** (JavaScript, C, C++, Python), but the DBMS understands **SQL** — there's a "wall" of languages between them. How does JS communicate with SQL?

**The solution — an Interface / API** (Application Program Interface): a program that converts host-language statements into SQL statements (or sends SQL wrapped in a package).

- **JDBC (Java Database Connectivity)** — API for **Java**.
- **ODBC (Open Database Connectivity)** — API for **C / C++** (made by Microsoft); imported like a package / include file.

**Real-world flow (the instructor's Spring Boot project example):**

1. Java app (back-end / server-side, using Spring + JDBC) writes the SQL query as a string/template (e.g., `INSERT INTO Candidate ... name, phone ...`).
2. JDBC sends the wrapped SQL query over to the **SQL server (the DBMS)**.
3. JDBC **transforms** it into an actual SQL query; the SQL server understands and runs it.
4. The reply comes back; JDBC **converts** the result into a Java construct/data structure; Java receives it.

```
  Java App (Spring Boot)
       │  writes SQL string: "INSERT INTO Candidate ..."
       ▼
  ┌─────────┐
  │  JDBC   │  ← translator/bridge between Java and SQL
  └─────────┘
       │  wraps & sends SQL over network
       ▼
  ┌────────────┐
  │ SQL Server │  ← understands SQL, executes query
  │  (DBMS)   │
  └────────────┘
       │  returns result set
       ▼
  JDBC converts result → Java data structure → Java App receives it
```

> **Interview point:** Know that if you write an app in a host language, you communicate with SQL via **JDBC (Java)** or **ODBC (C/C++)** — the interface acts as a layer/translator.

---

## 9. DBA — Database Administrator

**Concept & Logic:**

- The logical schema is the most important, so we need a person — the **DBA** — who has **central control** of **both** the data (the whole DB schema) **and** the programs/APIs that access the data.
- **Example (Aadhaar):** India's Aadhaar DB is used by many departments (Income Tax, EPFO, Home Ministry) — each with a different view schema. The DBA knows which programs/APIs access the data and what the data is. The DBA has central control of the entire DB.
- **DBA mostly works at the Logical Level** (also handles physical/internal storage). End users work at the View level.

### Functions of DBA (interview)

1. **Schema Definition** — define the schema (Student's attributes, storage structure, how things look at the physical level).
2. **Schema & Organization Modification** — handle any change in physical schema **or** logical schema. _Example:_ the school committee says "also store the student's Aadhaar info" → add a new column/attribute (Aadhaar) to the Student schema → logical schema & organization change; inter-dependencies (Student↔Course, Student↔Professor) may also change. DBA handles all this.
3. **Authorization Control** — who can access what data (like only authorized people accessing the DB). DBA controls who sees what.
4. **Routine Maintenance:**
    - **Periodic backups** — keep copies on multiple servers (if one DB server crashes, you don't lose the whole DB collected over years).
    - **Security patches** — apply latest patches so hackers can't exploit vulnerabilities (e.g., dark-web access to Aadhaar / NIC data).
    - **Upgrades** — e.g., changing the physical schema/design to make data access faster.

```
  DBA's role at a glance:

  ┌─────────────────────────────────────────────┐
  │                    DBA                      │
  │  ┌────────────┐  ┌──────────────────────┐   │
  │  │   Schema   │  │  Authorization       │   │
  │  │ Definition │  │  (who sees what)     │   │
  │  └────────────┘  └──────────────────────┘   │
  │  ┌────────────┐  ┌──────────────────────┐   │
  │  │  Schema    │  │  Routine Maintenance │   │
  │  │ Modification│ │  (backup, patches,   │   │
  │  └────────────┘  │   upgrades)          │   │
  │                  └──────────────────────┘   │
  └─────────────────────────────────────────────┘
         ▼ works mostly at the Logical Level
```

---

## 10. DBMS Application Architecture (1-Tier, 2-Tier, 3-Tier)

**Two key terms:**

- **Client Machine** — where the end user sits and works (e.g., you browsing YouTube).
- **Server Machine** — where the actual DB / DBMS system runs.

The architecture is defined by **how these machines are arranged** (where the DBMS server sits vs the client).

### 10.1 One-Tier Architecture (Tier-1)

- **Client, Server, and Database are all on the SAME machine.**
- Example: building a web app — you first run a **localhost** server inside your own computer. You are the client, the localhost server is yours, and the DB files are on your PC → all three on one PC.
- Example: learning SQL — the end user (you), the DBMS server, and the DB files are all on your own PC.

```
  Tier-1:  [Your PC]
           ┌─────────────────────────┐
           │  Client (browser/app)   │
           │  DBMS Server            │  ← all on one machine
           │  Database files         │
           └─────────────────────────┘
  Use case: learning SQL locally, localhost development
```

### 10.2 Two-Tier Architecture (Tier-2)

- The **client machine invokes DB functions at the server end through queries**.
- The host application writes **query statements** and sends them **over the network** via **JDBC / ODBC** interfaces.
- The **DBMS sits on the server** (remote PC); the client sends wrapped **SQL queries** directly to it; the DBMS understands and returns output.
- → System split into **two parts**: Client and Server, communicating via SQL queries (direct DB call over the network).
- Suitable for, e.g., a home/LAN system used by a few family devices (security isn't a big concern there).

```
  Tier-2:
  [Client Machine]  ──── SQL query (JDBC/ODBC) ────►  [Server Machine]
  App (Java/C++)                                        DBMS + Database
                    ◄──── result ────────────────────
  Use case: LAN, small internal apps, low-security scenarios
```

### 10.3 Three-Tier Architecture (Tier-3)

- Used for **World-Wide-Web / large-scale** applications (e.g., Amazon serving billions). The app is split into **three logical components**:
    1. **Client Machine** (just the front end)
    2. **Application Server**
    3. **Database System**
- **No direct DB call from the client.** Flow: **Client → Application Server → DB → Application Server → (beautified) → Client.** The client communicates with the app server (via an interface); the app server then calls the DB, retrieves data, modifies/beautifies it, and returns it to the client.

```
  Tier-3:
  [Client]  ──request──►  [App Server]  ──SQL──►  [DB Server]
  Browser                  (business logic,         DBMS +
  Mobile app               validation,              Database
                           beautification)
            ◄──response──               ◄──data──

  Use case: Amazon, YouTube, any large-scale internet application
```

**Advantages of Three-Tier:**

- **Scalability ↑** — split into 3 logical components; you can create **many application servers** and many clients → distributed application serving billions.
- **Data Integrity maintained** — the app server is a **middle layer**. A malicious/careless user could send a vulnerable/buggy SQL query that corrupts the DB; in Tier-2 that could destroy the whole DB. In Tier-3 the **app server filters out** such harmful queries at its layer.
- **Security maintained** — the DB is not accessed directly; a "guard"/interface sits in the middle, keeping the DB consistent and secure.

> **Choosing architecture:** Tier-1 → app only you use on your own computer. Tier-2 → e.g., LAN/home use (few users, low security concern). Tier-3 → large WWW apps (need app server for scalability, data integrity, and security, since you don't know what kind of user sits in which corner of the world).

---

## Lecture Recap (as stated in the video)

Most important: **Three-Schema Architecture** (end goal = multiple users access same data with personalized views via abstraction), **Instance**, **Schema**, **Data Models** (covered next), **DB Languages**, and **how application programs access the DB**.

> **Next lecture:** ER Model (Entity-Relationship Model) — a very powerful data model.