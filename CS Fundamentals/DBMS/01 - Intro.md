# What is DBMS? (Data, Information, Database & File System vs DBMS)

## 1. Data

**Concept & Logic:**

- In computer terms, everything you see — text, image, document, figure, symbol — is ultimately stored in the form of **bits and bytes**.
- **Data = a collection of raw bytes** (an unorganized collection of raw facts/details).
- **Example (image):** an image is a grid of pixels; each pixel has a value. An 8-bit image has 2⁸ = 256 different color shades. So an image is just a collection of bytes. Integers (1, 2, 3, 4, 5) are also a collection of bytes.

```
Everything on a computer, at its core:

  Text    →  bits & bytes
  Image   →  bits & bytes   } all just raw bytes
  Video   →  bits & bytes
  Numbers →  bits & bytes
```

**Key principle:** Raw data **has no meaning by itself**.

- Example: a column of numbers like `2.16, 2.79, 112, 135, 61, 66, ...` means nothing on its own.
- The moment you label the columns — **Height, Weight, BMI** — your mind **interprets / processes** it and extracts meaning ("these are persons; person 1's weight, height, and resulting BMI...").

```
Without labels (just raw data):
  2.16   112   61
  2.79   135   66
   ...   ...   ...
  → Three columns of numbers — completely meaningless on their own.
    You cannot tell if these are temperatures, prices, scores, or anything else.

With labels (processed — mind now interprets) -> *information*
  Height(m) | Weight(kg) | BMI
  ----------|------------|----
    2.16    |    112     |  61   ← these are the exact numbers from the example;
    2.79    |    135     |  66     once labelled, you read: person 1 is 2.16m tall,
    ...     |    ...     | ...     112kg, BMI 61. The numbers didn't change — only
                                   the label gave them meaning.
  → NOW it means something: these are persons' physical stats
```

> **Data, measured in bits and bytes, does NOT have meaning unless processed.**

---

## 2. Information

**Concept & Logic:**

- **Information = processed data.** Data → (process / interpret) → Information.

```css
  RAW DATA  ──── [Processing / Analysis] ────►  INFORMATION  ────►  DECISION
```

**Example A — Social media marketing:**

- A business gives its posts to a marketing company that uploads them on FB, Instagram, LinkedIn.
- Raw likes counts (e.g., FB post = some likes, Instagram post = 800 likes, LinkedIn post = 2 likes) are **data** (4-byte integers stored in a file).
- After **processing** this likes data → conclusion: _"maximum engagement is on Instagram → I should focus on Instagram."_ That conclusion is **information**.

```
  Platform     Likes (Data)
  ----------   ------------
  Facebook     200
  Instagram    800           ──► Processing ──► "Instagram = highest engagement"
  LinkedIn     2                                 = INFORMATION
                                                 ──► Decision: focus on Instagram
```

**Example B — Amazon feedback:**

- Customer feedback is raw **text** ("I love this product" / "I hate this product") → this is **data**.
- Amazon runs **sentiment analysis** on it → finds whether feedback is positive/negative, which **age group** likes it, customer rating, recommendation → this is **information**.
- Stakeholders then do **decision making** (which age group to show the product to, where to run ads, etc.).

**Example C — Census/locality data:**

- You have census data of your locality: how many members in each family, age group, gender.
- Process it → information: _"There are 100 senior citizens in this area."_
- Decision making from that information: _"I need to provide 100 senior citizen-related services here."_

**Why this matters:** "**Data is the new oil.**" Earlier oil (crude/petrol) was valuable; now data is — because with data you can grow a business, estimate what people in an area like, what food/clothes to sell, etc.

> **Data acts as a raw material.** Raw data is useless **unless it is processed** into information, and information enables **decision making**.

---

## 3. Data vs Information (Interview Table)

|Data|Information|
|---|---|
|Collection of facts|Puts those facts into **context**|
|Raw, unorganized|Organized|
|Insignificant for decision making by itself|You **can make decisions** based on it|
|Measured in bits & bytes|Result of processing data|

**Restaurant example (why data matters to businesses):**

- Person A and Person B both open restaurants with 10 dishes.
- **Person A** takes no feedback (no data collection). **Person B** takes feedback (collects data).
- Person B learns only 5 dishes sell well → orders **less raw material** for the rest → optimizes/controls cost.
- Person B also learns (from data) that _momos_ became very famous → markets them heavily.
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

- **Quantitative** — numerical form (weight, cost of an item).
- **Qualitative** — descriptive (name, gender, hair color of a person); no numericals.

---

## 4. Database

**Concept & Logic:**

- A **database** is an electronic place/system where data (in the form of bits & bytes) is stored **in a way that it can be easily accessed, managed, and updated**.
- It should NOT be encrypted/complicated such that accessing it takes a long time or needs advanced decryption each time.

```
  Database = Organized, electronic storage of data

  [Data on disk]
       ├── easily accessible
       ├── easily managed
       └── easily updated
```

> Analogy with OS: the OS manages **resources**; the **DBMS manages data**.

---

## 5. DBMS (Database Management System)

**Concept & Logic:**

- A **DBMS is software** — a **set of programs** that help me **access** data: i.e., **add, update, delete, and access** data.
- It also stores a **method to create the database** (how to store the data).

> **Definition:** _"DBMS is a collection of inter-related data and a set of programs to access that data."_

**Two parts of DBMS:**

1. **DB** — where data is stored.
2. **Set of programs** — to access / add / update / delete the data.

**Primary goal of a DBMS:** store data properly and **retrieve it efficiently and conveniently** (not taking hours to store or retrieve).

**Where it sits:** DBMS acts like an **interface** between the database (stored on disk) and the application programs / users that want to access the data. In modern systems (MySQL, Oracle), both the storage method and access method are provided together.

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

- A bank offers a **Saving Account** facility. A programmer is hired to build functions on a file system:
    1. Debit & Credit
    2. Add new account
    3. Find balance
    4. Generate monthly statements
- Each new account / debit-credit / balance is stored in separate files → many files generated.
- After 10 years the bank wants a **Current Account** feature (new concept). A new programmer must write **new files & new programs** (current account has no interest; saving account credits quarterly interest, etc.). Every change requires hiring a programmer and writing new code.

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

1. **Data Redundancy & Inconsistency** — Same person opens a Current Account 10 years after a Saving Account. A new programmer creates a separate new file → same person's details stored in **two places** = **Redundancy** (disk resource wasted). If the person updates address in the Saving Account file but forgets the Current Account file → same person has two different addresses = **Inconsistency**. Fixing this requires writing extra programs every time.

```
  saving_account.dat         current_account.dat
  ┌─────────────────┐        ┌─────────────────┐
  │ Rahul           │        │ Rahul           │
  │ 12 MG Road      │        │ 45 Park Street  │  ← different address!
  │ ...             │        │ ...             │
  └─────────────────┘        └─────────────────┘
        REDUNDANCY  ──────────────►  INCONSISTENCY
```

2. **Difficulty in Accessing Data** — A new request (e.g., "list all customers in PIN code 1001") that wasn't anticipated requires writing a **new program** each time (list → sort → filter). Not quick, not efficient. In SQL you just write a query and get the answer instantly.
    
3. **Data Isolation** — Different programmers use **different file formats** (e.g., `.dat` vs `.txt`). Same data sits in different formats across files → very difficult to retrieve and combine.
    
4. **Integrity Problems** — Consistency constraints (e.g., "current account balance can't go below ₹10,000") are manual code checks. If the constraint changes (RBI says minimum is now ₹20,000) → must hire a programmer to re-code it.
    
5. **Atomicity Problems** — A transaction must be **atomic** (all-or-nothing, no break in the middle): if money is debited from one account it **must** be credited to the other. If debit happened but credit didn't → money is lost. Very difficult to maintain in a file system; DBMS provides atomicity.
    

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

6. **Concurrent-Access Anomalies** — Multiple simultaneous operations on the same account (phone + debit card withdrawing at the same time) need **locks** so one request runs at a time. Managing concurrency in a file system is entirely manual.

```
  Account balance: ₹1000

  Request 1 (phone app):   reads ₹1000, decides to withdraw ₹800
  Request 2 (debit card):  reads ₹1000 (BEFORE R1 has written back), decides to withdraw ₹800

  Both see ₹1000 → both pass the "sufficient funds" check → both succeed
  R1 writes: ₹1000 - ₹800 = ₹200
  R2 writes: ₹1000 - ₹800 = ₹200  ← overwrites R1's result!
  Final balance: ₹200   (but ₹1600 was withdrawn — bank loses ₹800)

  Correct result should have been: 2nd withdrawal rejected (insufficient funds after 1st).
  DBMS uses locks so only one request runs at a time → anomaly prevented.
```

7. **Security Problems** — You don't want every user to access all data; some is confidential, some restricted, some public. Enforcing access rights in a file system requires writing your own programs each time — the biggest problem.

> **Summary:** Everything _can_ be done with a file system, but it requires huge manual effort (writing programs and checks for every case). As data volume exploded with the internet, a dedicated software — the **DBMS** — was built. DBMS provides (a) **methods to create the database** (store raw bytes) and (b) **methods to manage/access** it (e.g., "give me the first 20 students" or "show students with marks above 90%") — i.e., constrained, efficient access.

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

**Interview tip from the video:** If asked _"Why use a DBMS?"_ → state these 7 points (disadvantages of file system = advantages of DBMS).

---

> **Next lecture:** DBMS Architecture, Abstraction, Three-Schema Architecture, Data Independence & DBA.