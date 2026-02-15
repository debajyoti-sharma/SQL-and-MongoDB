# SQL and MongoDB Guide

A concise reference to SQL and MongoDB at basic and intermediate levels, with examples and explanations.

---

## Part 1: SQL

### Data model

SQL databases store data in **tables**. Each table has named **columns** (with types such as `INTEGER`, `VARCHAR(n)`, `DATE`, `DECIMAL`) and **rows** (records). A **primary key** uniquely identifies each row; it is often a single column (e.g. `id`) and is used to link tables.

```sql
CREATE TABLE users (
  id INTEGER PRIMARY KEY,
  name VARCHAR(100),
  created_at DATE
);

CREATE TABLE orders (
  id INTEGER PRIMARY KEY,
  user_id INTEGER,
  total DECIMAL(10,2),
  FOREIGN KEY (user_id) REFERENCES users(id)
);
```

### SELECT: reading data

`SELECT` specifies which columns to return. Use `*` for all columns. `DISTINCT` removes duplicate rows. Column aliases rename output columns with `AS`.

```sql
SELECT * FROM users;
SELECT name, created_at FROM users;
SELECT DISTINCT name FROM users;
SELECT name AS user_name, id AS user_id FROM users;
```

### Filtering with WHERE

`WHERE` restricts rows by conditions. Use comparison operators (`=`, `<>`, `<`, `>`, `<=`, `>=`), logical operators (`AND`, `OR`, `NOT`), and special predicates.

```sql
SELECT * FROM users WHERE id = 1;
SELECT * FROM users WHERE id > 1 AND name IS NOT NULL;
SELECT * FROM users WHERE name IN ('Alice', 'Bob');
SELECT * FROM users WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31';
SELECT * FROM users WHERE name LIKE 'A%';
SELECT * FROM users WHERE phone IS NULL;
```

- `IN (list)` matches any value in the list.
- `BETWEEN a AND b` is inclusive.
- `LIKE` uses `%` (any string) and `_` (one character).
- `IS NULL` / `IS NOT NULL` test for nulls.

### Sorting and limiting

`ORDER BY` sorts results; add `ASC` (default) or `DESC`. `LIMIT n` returns at most `n` rows; `OFFSET m` skips the first `m` rows. Many databases use `LIMIT`/`OFFSET` (e.g. PostgreSQL, SQLite, MySQL); SQL Server uses `TOP n` or `OFFSET ... FETCH`.

```sql
SELECT * FROM users ORDER BY name ASC;
SELECT * FROM users ORDER BY created_at DESC LIMIT 10;
SELECT * FROM users ORDER BY id LIMIT 5 OFFSET 10;
```

### Aggregation

Aggregate functions reduce many rows to one value: `COUNT(*)`, `SUM(column)`, `AVG(column)`, `MIN(column)`, `MAX(column)`. `GROUP BY` computes these per group. `HAVING` filters groups after aggregation (unlike `WHERE`, which filters rows before aggregation).

```sql
SELECT COUNT(*) FROM users;
SELECT country, COUNT(*) AS user_count FROM users GROUP BY country;
SELECT category, SUM(amount) AS total FROM orders GROUP BY category
  HAVING SUM(amount) > 1000;
```

### Joins

Joins combine rows from two (or more) tables. **INNER JOIN** returns only rows that match in both tables. **LEFT JOIN** returns all rows from the left table and matching rows from the right; missing right-side columns are NULL.

Example tables:

| id | name  |
|----|-------|
| 1  | Alice |
| 2  | Bob   |

| id | user_id | total |
|----|---------|-------|
| 1  | 1       | 50.00 |
| 2  | 1       | 30.00 |
| 3  | 3       | 20.00 |

```sql
SELECT u.name, o.total
FROM users u
INNER JOIN orders o ON u.id = o.user_id;
-- Returns: Alice 50.00, Alice 30.00 (Bob and user_id 3 have no match)

SELECT u.name, o.total
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;
-- Returns: Alice 50.00, Alice 30.00, Bob NULL (all users; Bob has no orders)
```

**RIGHT JOIN** keeps all rows from the right table; **FULL OUTER JOIN** keeps all rows from both and fills NULLs where there is no match. Not all databases support them (e.g. MySQL has no FULL OUTER JOIN).

### Subqueries

A **scalar subquery** returns one row and one column; it can appear where a single value is expected.

```sql
SELECT name FROM users WHERE id = (SELECT user_id FROM orders ORDER BY total DESC LIMIT 1);
```

**IN (subquery)** tests whether a value is in the subquery’s result set.

```sql
SELECT * FROM users WHERE id IN (SELECT user_id FROM orders WHERE total > 100);
```

A subquery in `FROM` acts as a **derived table**; it must have an alias.

```sql
SELECT avg_total FROM (
  SELECT user_id, AVG(total) AS avg_total FROM orders GROUP BY user_id
) AS sub WHERE avg_total > 50;
```

### Set operations

`UNION` combines two queries’ rows and removes duplicates; `UNION ALL` keeps duplicates. Column count and types must match.

```sql
SELECT name FROM users WHERE country = 'US'
UNION
SELECT name FROM archived_users WHERE country = 'US';
```

### Indexes

An **index** is a structure that speeds up lookups and sorts on one or more columns. Create one with `CREATE INDEX`. Indexes help `WHERE`, `JOIN`, and `ORDER BY` on the indexed columns but add cost on writes (insert/update/delete). Too many indexes can slow writes and use extra storage; create them for columns you filter or sort on often.

```sql
CREATE INDEX idx_users_name ON users(name);
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_user_total ON orders(user_id, total);
```

### Transactions

A **transaction** groups several statements so they either all take effect or none do. You start with `BEGIN` (or `START TRANSACTION`), then run statements, and end with `COMMIT` (save) or `ROLLBACK` (undo). Transactions provide ACID: Atomicity (all or nothing), Consistency (rules preserved), Isolation (concurrent transactions don’t see uncommitted changes in a way that breaks the rules), Durability (committed data survives crashes).

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
-- If something goes wrong before COMMIT, use ROLLBACK; instead.
```

---

## Part 2: MongoDB

### Data model

MongoDB stores data as **documents** in **collections**. Documents are JSON-like structures (BSON) with field–value pairs. Each document usually has an `_id` (unique per collection); if you omit it, MongoDB generates an ObjectId. Common field types include string, number, boolean, array, embedded document, and date.

```javascript
// Example document in a collection "users"
{
  _id: ObjectId("507f1f77bcf86cd799439011"),
  name: "Alice",
  age: 30,
  tags: ["admin", "editor"],
  address: { city: "Boston", zip: "02101" },
  created_at: ISODate("2024-01-15T10:00:00Z")
}
```

### CRUD: insert

`insertOne` adds a single document; `insertMany` adds several. You can omit `_id` to let MongoDB generate it.

```javascript
db.users.insertOne({ name: "Bob", age: 25 });

db.users.insertMany([
  { name: "Carol", age: 28 },
  { name: "Dave", age: 32 }
]);
```

### CRUD: find

`find(filter, projection)` returns documents matching the filter. An empty filter `{}` returns all documents. The second argument is an optional **projection**: which fields to include (1) or exclude (0).

```javascript
db.users.find({});
db.users.find({ age: 30 });
db.users.find({ age: { $gte: 25 } }, { name: 1, age: 1 });
db.users.find({ age: 25 }, { name: 0 });
```

In projection, you cannot mix include and exclude (except for `_id`). `_id` is included by default; set `_id: 0` to exclude it.

### Query operators

Operators are used inside the filter object, often with a field and a value.

| Operator | Meaning   | Example |
|----------|-----------|---------|
| `$eq`    | equals    | `{ age: { $eq: 30 } }` or `{ age: 30 }` |
| `$ne`    | not equal | `{ status: { $ne: "inactive" } }` |
| `$gt`, `$gte`, `$lt`, `$lte` | greater/less than (or equal) | `{ age: { $gte: 18, $lte: 65 } }` |
| `$in`    | in list   | `{ role: { $in: ["admin", "editor"] } }` |
| `$exists`| field exists | `{ phone: { $exists: true } }` |
| `$regex`| pattern   | `{ name: { $regex: /^Al/ } }` |

```javascript
db.users.find({ age: { $gt: 25 } });
db.users.find({ tags: { $in: ["admin"] } });
db.users.find({ name: { $regex: /son$/ } });
```

### CRUD: update

`updateOne` updates the first document that matches the filter; `updateMany` updates all matches. The update document usually uses **update operators** such as `$set` (set fields) and `$unset` (remove fields).

```javascript
db.users.updateOne(
  { name: "Bob" },
  { $set: { age: 26, updated_at: new Date() } }
);

db.users.updateMany(
  { role: "guest" },
  { $set: { role: "user" } }
);

db.users.updateOne(
  { name: "Bob" },
  { $unset: { nickname: "" } }
);
```

### CRUD: delete

`deleteOne` removes the first matching document; `deleteMany` removes all matches.

```javascript
db.users.deleteOne({ name: "Bob" });
db.users.deleteMany({ status: "archived" });
```

### Sorting and limiting

Chain `.sort()`, `.limit()`, and `.skip()` on the cursor returned by `find()`. In `sort`, use `1` for ascending and `-1` for descending.

```javascript
db.users.find({}).sort({ age: 1 });
db.users.find({}).sort({ created_at: -1 }).limit(10);
db.users.find({}).sort({ name: 1 }).skip(20).limit(10);
```

### Embedded vs referenced data

**Embedded documents** store related data inside a single document. This works well when the related set is small, is read together with the parent, and is not shared across many parents (e.g. a user’s addresses).

```javascript
{
  _id: 1,
  name: "Alice",
  addresses: [
    { city: "Boston", zip: "02101" },
    { city: "NYC", zip: "10001" }
  ]
}
```

**Referenced data** stores a link (e.g. an id) in one document and the full data in another collection. Use this when the related set is large, shared by many parents, or updated independently (e.g. orders referencing products).

```javascript
// orders collection
{ _id: 1, user_id: 101, product_id: 5, qty: 2 }

// users collection
{ _id: 101, name: "Alice" }

// products collection
{ _id: 5, name: "Widget", price: 9.99 }
```

### Aggregation pipeline

The aggregation pipeline processes documents through stages. Each stage takes the output of the previous one. Common stages: `$match` (filter), `$group` (aggregate by key), `$project` (reshape fields), `$sort`, `$limit`.

```javascript
// Sales total by category, top 5 categories
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $group: { _id: "$category", total: { $sum: "$amount" }, count: { $sum: 1 } } },
  { $sort: { total: -1 } },
  { $limit: 5 },
  { $project: { category: "$_id", total: 1, count: 1, _id: 0 } }
]);
```

Inside `$group`, `_id` is the grouping key (e.g. `"$category"`). Accumulators like `$sum`, `$avg`, `$min`, `$max` compute per group.

### Indexes

Indexes speed up queries and sorts. Create a single-field index or a compound index (order of fields matters for sort and range queries).

```javascript
db.users.createIndex({ name: 1 });
db.users.createIndex({ age: -1 });
db.orders.createIndex({ user_id: 1, created_at: -1 });
```

Use indexes for fields that appear in filters or in `sort()`. Too many indexes increase write cost and storage; create them for common query patterns.

### Update operators for arrays and numbers

`$inc` increments (or decrements) a numeric field. `$push` appends to an array; `$addToSet` appends only if the value is not already in the set; `$pull` removes matching elements.

```javascript
db.products.updateOne(
  { _id: 5 },
  { $inc: { stock: -2 }, $push: { recent_buyers: "user101" } }
);

db.users.updateOne(
  { name: "Alice" },
  { $addToSet: { tags: "vip" } }
);

db.users.updateOne(
  { name: "Alice" },
  { $pull: { tags: "trial" } }
);
```

---

*This guide covers core SQL and MongoDB concepts at a basic and intermediate level. Syntax may vary slightly by database or driver; refer to your engine’s documentation for details.*
