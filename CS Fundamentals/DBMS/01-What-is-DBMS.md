## 1. Data

- In computer terms, everything (text, image, document, figure, symbol) is ultimately stored as **bits and bytes**.
- **Data = a collection of raw bytes** (an unorganized collection of raw facts/details).
- **Example (image):** an image is a grid of pixels; each pixel has a value. An 8-bit image has 2⁸ = 256 color shades. An image is a collection of bytes. Integers (1, 2, 3, 4, 5) are also a collection of bytes.

```
Everything on a computer, at its core:

  Text    →  bits & bytes
  Image   →  bits & bytes   } all just raw bytes
  Video   →  bits & bytes
  Numbers →  bits & bytes
```

- **Key principle:** Raw data **has no meaning by itself**.
  - A column of numbers like `2.16, 2.79, 112, 135, 61, 66, ...` means nothing on its own.
  - Labelling the columns — **Height, Weight, BMI** — interprets/processes the data and extracts meaning.

```
Without labels (just raw data):
  2.16   112   61
  2.79   135   66
   ...   ...   ...
  → Three columns of numbers — meaningless on their own.
    Cannot tell if these are temperatures, prices, scores, etc.

With labels (processed):
  Height(m) | Weight(kg) | BMI
  ----------|------------|----
    2.16    |    112     |  61
    2.79    |    135     |  66
    ...     |    ...     | ...
  → Now meaningful: persons' physical stats.
    The numbers didn't change — only the label gave them meaning.
```

> **Data, measured in bits and bytes, does NOT have meaning unless processed.**

---

## 2. Information

- **Information = processed data.** Data → (process / interpret) → Information.

```
  RAW DATA  ──── [Processing / Analysis] ────►  INFORMATION  ────►  DECISION
```

**Example A — Social media marketing:**
- Raw like counts (e.g., Facebook = 200, Instagram = 800, LinkedIn = 2) are **data** (4-byte integers stored in a file).
- After **processing** → conclusion: *"maximum engagement is on Instagram → focus on Instagram."* That conclusion is **information**.

```
  Platform     Likes (Data)
  ----------   ------------
  Facebook     200
  Instagram    800           ──► Processing ──► "Instagram = highest engagement"
  LinkedIn     2                                 = INFORMATION
                                                 ──► Decision: focus on Instagram
```

**Example B — Amazon feedback:**
- Customer feedback is raw **text** ("I love this product" / "I hate this product") = **data**.
- **Sentiment analysis** finds whether feedback is positive/negative, which **age group** likes it, customer rating, recommendation = **information**.
- Stakeholders then do **decision making** (which age group to show the product to, where to run ads, etc.).

**Example C — Census/locality data:**
- Census data: members in each family, age group, gender.
- Process it → information, e.g.: *"100 senior citizens in this area"*, *"sex ratio is 1.1"*, *"100 newborn babies."*
- Decision making: *"provide 100 senior citizen-related services here."*

- **Data is the new oil.** With data you can grow a business and estimate what people in an area like.

> **Data acts as raw material.** Raw data is useless **unless processed** into information, and information enables **decision making**.

---

## 3. Data vs Information (Table)

| Data | Information |
|------|-------------|
| Collection of facts | Puts those facts into **context** |
| Raw, unorganized | Organized |
| Insignificant for decision making by itself | You **can make decisions** based on it |
| Measured in bits & bytes | Result of processing data |
| Does **not** depend on information | **Depends on** data (information is extracted from data) |
| Typically comes as graphs, numbers, figures, or statistics | Typically presented through words, language, thoughts, and ideas |

**Restaurant example (why data matters to businesses):**
- Person A and Person B both open restaurants with 10 dishes.
- **Person A** takes no feedback (no data collection). **Person B** takes feedback (collects data).
- Person B learns only 5 dishes sell well → orders **less raw material** for the rest → optimizes/controls cost.
- Person B also learns *momos* became famous → markets them heavily.
- Result: Person B's profitability and sales go far ahead; Person A may fail.

```
Person A (no data):          Person B (collects data):
  10 dishes, no feedback       10 dishes, collects feedback
  orders same raw material     learns: only 5 dishes sell
  wastes on unsold dishes      orders less for 5 slow dishes
  no targeted marketing        markets "momos" heavily
        ▼                              ▼
   Struggles / fails           Profitable & growing
```

### Types of Data
- **Quantitative** — numerical form (weight, volume, cost of an item).
- **Qualitative** — descriptive (name, gender, hair color); no numericals.

---

## 4. Database

- A **database** is an electronic place/system where data (in bits & bytes) is stored **so it can be easily accessed, managed, and updated**.
- It should NOT be encrypted/complicated such that accessing it takes a long time or needs advanced decryption each time.

```
  Database = Organized, electronic storage of data

  [Data on disk]
       ├── easily accessible
       ├── easily managed
       └── easily updated
```

- OS manages **resources**; the **DBMS manages data**.

---

## 5. DBMS (Database Management System)

- A **DBMS is software** — a **set of programs** that help **access** data: **add, update, delete, and access** data.
- It also stores a **method to create the database** (how to store the data).

> **Definition:** *"DBMS is a collection of inter-related data and a set of programs to access that data."* The collection of data (the **database**) contains information **relevant to an enterprise**.

**Two parts of DBMS:**
1. **DB** — where data is stored.
2. **Set of programs** — to access / add / update / delete the data.

**Primary goal of a DBMS:** store data properly and **retrieve it efficiently and conveniently**.

**Where it sits:** DBMS acts as an **interface** between the database (on disk) and the application programs / users. In modern systems (MySQL, Oracle), both storage method and access method are provided together.

```
  [User / Application]
          │
          ▼
    ┌───────────┐
    │   DBMS    │  ← software layer (MySQL, Oracle, PostgreSQL …)
    │ (programs)│     • add / update / delete / retrieve
    └───────────┘
          │
          ▼
    ┌───────────┐
    │ Database  │  ← actual data stored on disk (bits & bytes)
    └───────────┘
```

---

## 6. Why Not Just Use a File System? (File System vs DBMS)

**Bank example (early systems, file-system based):**
- A bank offers a **Saving Account** facility. A programmer builds functions on a file system:
  1. Debit & Credit
  2. Add new account
  3. Find balance
  4. Generate monthly statements
- Each new account / debit-credit / balance is stored in separate files → many files generated.
- Later the bank wants a **Current Account** feature. A new programmer must write **new files & new programs** (current account has no interest; saving account credits quarterly interest). Every change requires hiring a programmer and writing new code.

```
File System approach (early banks):

  saving_account.dat   current_account.dat   transactions.dat
       │                      │                     │
  hand-written             hand-written         hand-written
  program A                program B             program C
  (debit/credit)           (new acct)            (balance)

  Problem: every new requirement = write a new program from scratch
```

### The 7 Major Disadvantages of File System (= 7 Advantages of DBMS)

1. **Data Redundancy & Inconsistency** — Same person opens a Current Account years after a Saving Account. A new file stores the same person's details in **two places** = **Redundancy** (disk wasted). If the person updates address in the Saving Account file but not the Current Account file → two different addresses = **Inconsistency**. Fixing this requires extra programs every time.

```
  saving_account.dat         current_account.dat
  ┌─────────────────┐        ┌─────────────────┐
  │ Rahul           │        │ Rahul           │
  │ 12 MG Road      │        │ 45 Park Street  │  ← different address!
  │ ...             │        │ ...             │
  └─────────────────┘        └─────────────────┘
        REDUNDANCY  ──────────────►  INCONSISTENCY
```

2. **Difficulty in Accessing Data** — A new request (e.g., "list all customers in PIN code 1001") that wasn't anticipated requires writing a **new program** each time (list → sort → filter). In SQL you just write a query and get the answer instantly.

3. **Data Isolation** — Different programmers use **different file formats** (e.g., `.dat` vs `.txt`). Same data in different formats across files → difficult to retrieve and combine.

4. **Integrity Problems** — Consistency constraints (e.g., "current account balance can't go below ₹10,000") are manual code checks. If the constraint changes (RBI says minimum is now ₹20,000) → must re-code it.

5. **Atomicity Problems** — A transaction must be **atomic** (all-or-nothing, no break in the middle): if money is debited from one account it **must** be credited to the other. If debit happened but credit didn't → money is lost. Difficult to maintain in a file system; DBMS provides atomicity.

```
  Transfer ₹500 from A → B

  File System risk:
    Step 1: Debit A  ✓  (A loses ₹500)
    Step 2: CRASH!   ✗  (B never credited)
            ──► ₹500 lost forever

  DBMS with Atomicity:
    Either BOTH steps succeed, or NEITHER happens.
    A crash mid-way → automatically rolled back.
```

6. **Concurrent-Access Anomalies** — Multiple simultaneous operations on the same account (phone + debit card withdrawing at the same time) need **locks** so one request runs at a time. Managing concurrency in a file system is entirely manual.

```
  Account balance: ₹1000

  Request 1 (phone app):   reads ₹1000, decides to withdraw ₹800
  Request 2 (debit card):  reads ₹1000 (BEFORE R1 has written back), decides to withdraw ₹800

  Both see ₹1000 → both pass the "sufficient funds" check → both succeed
  R1 writes: ₹1000 - ₹800 = ₹200
  R2 writes: ₹1000 - ₹800 = ₹200  ← overwrites R1's result!
  Final balance: ₹200   (but ₹1600 was withdrawn — bank loses ₹800)

  Correct result: 2nd withdrawal rejected (insufficient funds after 1st).
  DBMS uses locks so only one request runs at a time → anomaly prevented.
```

7. **Security Problems** — Not every user should access all data; some is confidential, restricted, or public. Enforcing access rights in a file system requires writing your own programs each time.

> **Summary:** Everything *can* be done with a file system, but it requires huge manual effort (writing programs and checks for every case). The **DBMS** provides (a) **methods to create the database** (store raw bytes) and (b) **methods to manage/access** it (e.g., "give me the first 20 students" or "show students with marks above 90%") — constrained, efficient access.

```
File System vs DBMS — quick comparison:

  Problem               File System                        DBMS
  ──────────────────    ──────────────────────────────     ─────────────────────────────────
  Redundancy &          Same data copied in many files;    Single source; updates propagate
  Inconsistency         updates must be done everywhere    automatically via constraints
  Difficulty accessing  Write a new program for each       One SQL query, answered instantly
  Data isolation        Different formats (.dat/.txt)      Single unified data model
  Integrity             Constraints coded manually         Constraints declared once, always
                        per program, easy to miss          checked on every operation
  Atomicity             No built-in rollback; crash        Transactions: all-or-nothing,
                        leaves data in broken state        auto-rollback on failure
  Concurrency           Manual locking logic per app       Built-in concurrency control
  Security              Hand-written access checks         Access control at the DBMS level
```

**Interview tip:** If asked *"Why use a DBMS?"* → state these 7 points (disadvantages of file system = advantages of DBMS).

---

> **Next lecture:** DBMS Architecture, Abstraction, Three-Schema Architecture, Data Independence & DBA.
