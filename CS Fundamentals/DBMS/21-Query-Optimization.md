## Query Optimization

---

## 1. Index Scan vs Table Scan (Full Seq Scan)

When you run a `SELECT ... WHERE ...`, the DB has two ways to find matching rows:

- **Table Scan (Seq Scan):** read every single block of the table from start to finish, check each row against the filter.
- **Index Scan:** walk the B-Tree index to locate exactly which blocks contain the matching rows, then fetch only those blocks.

| | Table Scan | Index Scan |
|---|---|---|
| **What it does** | Reads every block of the table sequentially | Traverses B-Tree to find matching rows, then fetches those blocks |
| **When it's used** | No index exists, or query returns a large fraction of rows | Index exists and the query is selective (few rows match) |
| **When it wins** | Returning > ~10–20% of rows | Returning a small fraction of rows |

**Why would the optimizer ever ignore an index?** Index scan does random I/O — it jumps to scattered blocks on disk. Sequential I/O (table scan) is much faster per block because the disk head doesn't have to seek. So if you're returning 100,000 scattered rows, 100,000 random seeks can be *slower* than one smooth sequential pass over the whole table. The optimizer estimates this and picks accordingly.

This is why **an index on a low-cardinality column** (e.g., `is_active` with only 2 values, or `status` with 3 values) is often useless — the query still returns too many rows for the index to help.

```css
INDEX SCAN vs TABLE SCAN — when each wins:

  Table: 1,000,000 orders

  WHERE status = 'pending'  (returns 10% = 100,000 rows)
    → 100,000 random disk seeks via index
    → vs one sequential pass of the whole table
    → sequential scan is cheaper → optimizer skips the index ❌

  WHERE order_id = 98765  (returns 1 row)
    → B-Tree lookup: 3–4 disk reads to find the one row
    → vs sequential scan of 1,000,000 rows
    → index scan wins by a huge margin ✅

  Rule: index wins when the query is selective (returns a small % of rows).
```

---

## 2. N+1 Query Problem

### What Is It?

The N+1 problem is an **application-level anti-pattern** where code runs **1 query** to fetch N parent records, then fires **N more queries** — one per row — to fetch related data. It's easy to write accidentally when using ORMs.

```css
N+1 QUERY PROBLEM:

  Code:
    users = db.query("SELECT * FROM users LIMIT 100")   ← 1 query, 100 users

    for user in users:
        orders = db.query(f"SELECT * FROM orders WHERE user_id = {user.id}")
        print(user.name, len(orders))

  What actually executes against the DB:
    Query 1:   SELECT * FROM users LIMIT 100
    Query 2:   SELECT * FROM orders WHERE user_id = 1
    Query 3:   SELECT * FROM orders WHERE user_id = 2
    ...
    Query 101: SELECT * FROM orders WHERE user_id = 100

  Total: 101 queries instead of 1 or 2.
  For 1000 users: 1001 queries.
  With 10ms network latency per query: 101 × 10ms ≈ 1 second just in round-trips.
```

Each individual query is fast. The problem is the **cumulative overhead** of 100 round-trips — especially when the DB is on a separate server.

### Fix 1 — JOIN (one query, everything together)

```sql
SELECT users.id, users.name, orders.id, orders.amount
FROM users
LEFT JOIN orders ON orders.user_id = users.id
WHERE users.id IN (...);
```

The DB does the join internally. One round-trip, all data returned together.

### Fix 2 — Batch IN (two queries, merge in app)

```sql
-- Step 1: fetch users
SELECT * FROM users LIMIT 100;
-- Step 1 gives you IDs: [1, 2, ..., 100]

-- Step 2: fetch ALL their orders in one query
SELECT * FROM orders WHERE user_id IN (1, 2, 3, ..., 100);
```

Two queries total instead of 101. You then match users to their orders in application memory. This is what ORMs call **eager loading** (`select_related` / `prefetch_related` in Django, `includes` in Rails).

```css
COMPARISON:

  N+1:         101 queries, 101 round-trips
  JOIN:          1 query,     1 round-trip   ← best when you need joined columns
  Batch IN:      2 queries,   2 round-trips  ← best when you want separate objects in app

  At 10ms latency:
    N+1:     101 × 10ms = ~1s
    JOIN/IN: ≤ 2 × 10ms = ~20ms
```

---

## 3. Query Optimization Techniques

### Avoid SELECT *

`SELECT *` fetches every column in the table — including large `TEXT` or `BLOB` columns you may never use. This means more data read from disk, more data sent over the network, and more memory used.

The less obvious cost: `SELECT *` **breaks covering indexes**. A covering index is when the index itself contains all the columns your query needs — the DB can answer the query entirely from the index without touching the main table. `SELECT *` always forces a table lookup, so covering indexes never kick in.

```sql
-- Bad: fetches all columns including a large 'description' blob
SELECT * FROM products WHERE category = 'electronics';

-- Good: only fetch what you actually need
SELECT id, name, price FROM products WHERE category = 'electronics';
-- If there's an index on (category, id, name, price) → answered from index alone, no table read
```

### Avoid Functions on Indexed Columns in WHERE

If you wrap an indexed column inside a function in your `WHERE` clause, the DB **cannot use the index**. The index stores the raw column values sorted — it has no idea what `YEAR(created_at)` would produce for each entry. So it has to compute the function for every row and check — full scan.

```sql
-- Bad: function wraps the column → index on created_at is useless
WHERE YEAR(created_at) = 2024

-- Good: express the same condition as a range on the raw column → index works
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'
```

Same problem applies to: `LOWER(email) = 'x'`, `DATE(created_at) = '2024-01-01'`, `LENGTH(name) > 5`, etc.

### Avoid Leading Wildcards in LIKE

```sql
WHERE name LIKE '%sharma'   -- ❌ full table scan — leading % means "any prefix"
WHERE name LIKE 'sharma%'   -- ✅ uses index — "starts with sharma" maps to a range in the B-Tree
```

A B-Tree index on `name` stores values in alphabetical order. `'sharma%'` is a contiguous range — the DB can jump to "sharma" and scan forward. `'%sharma'` could match anywhere, so the DB has to check every single row.

### Use EXISTS Instead of COUNT for Existence Checks

When you just want to know *if* any matching rows exist, `COUNT(*)` is wasteful — it scans and counts all matching rows just to tell you the number is > 0. `EXISTS` stops the moment it finds the first match.

```sql
-- Bad: counts all matching rows, even if there are 10,000
SELECT * FROM users
WHERE (SELECT COUNT(*) FROM orders WHERE user_id = users.id) > 0;

-- Good: stops at the first matching row
SELECT * FROM users
WHERE EXISTS (SELECT 1 FROM orders WHERE user_id = users.id);
```

### Keyset Pagination Instead of OFFSET

`LIMIT 20 OFFSET 10000` sounds reasonable but is slow — the DB has to **fetch and discard** 10,000 rows before it can return your 20. The higher the page, the worse it gets.

Keyset pagination (a.k.a. cursor-based pagination) avoids this by remembering the last seen ID and using `WHERE id > last_id` — the index jumps directly to that position.

```sql
-- Bad: fetches and throws away 10,000 rows on every "next page" click
SELECT * FROM orders ORDER BY id LIMIT 20 OFFSET 10000;

-- Good: uses the index to jump directly past the last seen row
SELECT * FROM orders WHERE id > 10000 ORDER BY id LIMIT 20;
-- "10000" is the id of the last row from the previous page
```

The tradeoff: keyset pagination doesn't support jumping to an arbitrary page number ("go to page 47"). It only supports next/previous. For most feeds and infinite-scroll UIs that's fine.

---

## 4. Quick Revision

| Concept | One-liner |
|---|---|
| Index Scan | Fast for selective queries; optimizer ignores it when too many rows match |
| Table Scan | Reads all rows sequentially — wins when returning a large fraction of the table |
| N+1 Problem | 1 query fetches N rows, then N more queries fetch related data — fix with JOIN or batch IN |
| Eager Loading | Fetch all related data upfront in bulk instead of one query per row |
| SELECT * | Fetches unused columns and prevents covering indexes — always name your columns |
| Function on indexed column | Forces full scan — rewrite as a range condition on the raw column |
| LIKE '%x' | Leading wildcard forces full scan — use trailing wildcard `'x%'` instead |
| EXISTS vs COUNT | EXISTS stops at first match; COUNT scans everything — use EXISTS for existence checks |
| OFFSET pagination | Discards rows it had to read — use keyset (`WHERE id > last_id`) for large pages |

> Eventual vs Strong Consistency → covered in [[18-CAP Theorem]]
