# Database Design & SQL

## 1. Normalization

### Why Normalize?
- Eliminate data redundancy
- Prevent update/delete anomalies
- Ensure data integrity

### 1NF (First Normal Form)
**Rule:** Each cell contains a single atomic value. No repeating groups.

```sql
-- BAD (violates 1NF)
CREATE TABLE orders (
    id INT PRIMARY KEY,
    customer_name VARCHAR(255),
    products TEXT  -- "Laptop, Mouse, Keyboard"
);

-- GOOD (1NF)
CREATE TABLE orders (
    id INT PRIMARY KEY,
    customer_name VARCHAR(255)
);

CREATE TABLE order_items (
    id INT PRIMARY KEY,
    order_id INT,
    product_name VARCHAR(255),
    quantity INT
);
```

### 2NF (Second Normal Form)
**Rule:** Must be in 1NF + no partial dependencies (non-key depends on PART of composite PK).

```sql
-- BAD (violates 2NF)
CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    quantity INT,
    product_name VARCHAR(255),  -- depends only on product_id
    order_date DATE,            -- depends only on order_id
    PRIMARY KEY (order_id, product_id)
);

-- GOOD (2NF)
CREATE TABLE orders (
    id INT PRIMARY KEY,
    order_date DATE
);

CREATE TABLE products (
    id INT PRIMARY KEY,
    name VARCHAR(255)
);

CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    quantity INT,
    PRIMARY KEY (order_id, product_id)
);
```

### 3NF (Third Normal Form)
**Rule:** Must be in 2NF + no transitive dependencies (non-key depends on another non-key).

```sql
-- BAD (violates 3NF)
CREATE TABLE employees (
    id INT PRIMARY KEY,
    name VARCHAR(255),
    department_id INT,
    department_name VARCHAR(255)  -- depends on department_id, not id!
);

-- GOOD (3NF)
CREATE TABLE departments (
    id INT PRIMARY KEY,
    name VARCHAR(255)
);

CREATE TABLE employees (
    id INT PRIMARY KEY,
    name VARCHAR(255),
    department_id INT,
    FOREIGN KEY (department_id) REFERENCES departments(id)
);
```

### When to Denormalize
- Read-heavy tables where JOINs are expensive
- Reporting/analytics queries
- Caching frequently computed values
- Example: adding `comment_count` to posts table

---

## 2. NULL Behavior (Interview Focus)

### NULL = NULL is NULL, Not TRUE

```sql
SELECT NULL = NULL;    -- NULL (not TRUE!)
SELECT NULL = 0;       -- NULL
SELECT NULL = 'foo';   -- NULL
SELECT NULL <> 0;      -- NULL
SELECT NULL > 0;       -- NULL
SELECT NULL < 0;       -- NULL

-- Always use IS NULL / IS NOT NULL
SELECT * FROM users WHERE deleted_at IS NULL;
SELECT * FROM users WHERE deleted_at IS NOT NULL;
```

### NULL in UNIQUE Columns

```sql
-- MySQL/PostgreSQL allow multiple NULLs in UNIQUE columns!
CREATE TABLE users (
    id INT PRIMARY KEY,
    email VARCHAR(255) UNIQUE
);

INSERT INTO users (id, email) VALUES (1, 'a@b.com');
INSERT INTO users (id, email) VALUES (2, 'a@b.com');  -- ERROR
INSERT INTO users (id, email) VALUES (3, NULL);        -- OK
INSERT INTO users (id, email) VALUES (4, NULL);        -- OK (multiple NULLs!)

-- PostgreSQL: NULLS NOT DISTINCT (PG 15+)
CREATE TABLE users (
    id INT PRIMARY KEY,
    email VARCHAR(255) UNIQUE NULLS NOT DISTINCT
);
-- Now only one NULL is allowed
```

### NULL in Indexes

```sql
-- B-tree indexes:
-- MySQL: NULL values ARE indexed
-- PostgreSQL: NULL values ARE indexed (but you can create partial index)

-- Create partial index (PostgreSQL)
CREATE INDEX idx_users_email ON users (email) WHERE email IS NOT NULL;
```

### NULL Functions

```sql
-- COALESCE: return first non-NULL value
SELECT COALESCE(nickname, username, 'Anonymous') AS display_name FROM users;

-- NULLIF: return NULL if two values are equal
SELECT NULLIF(price, 0);  -- returns NULL if price is 0

-- IFNULL (MySQL) / NVL (Oracle)
SELECT IFNULL(email, 'no-email@example.com') FROM users;

-- ISNULL (SQL Server)
SELECT ISNULL(email, 'no-email@example.com') FROM users;

-- CASE with NULL
SELECT CASE
    WHEN email IS NULL THEN 'No email'
    ELSE email
END FROM users;
```

### NULL in Aggregate Functions

```sql
-- COUNT(*) counts all rows, COUNT(column) ignores NULLs
SELECT COUNT(*) FROM users;          -- counts all
SELECT COUNT(email) FROM users;      -- ignores NULL emails

-- SUM/AVG ignore NULLs
SELECT SUM(amount) FROM orders;  -- NULLs are ignored
SELECT AVG(amount) FROM orders;  -- only averages non-NULL values
```

---

## 3. Constraints

### PRIMARY KEY

```sql
-- Single column
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(255)
);

-- Composite primary key
CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    quantity INT,
    PRIMARY KEY (order_id, product_id)
);

-- Auto increment (MySQL)
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY
);

-- Serial (PostgreSQL)
CREATE TABLE users (
    id SERIAL PRIMARY KEY
);
```

### UNIQUE Constraints

```sql
-- Single column unique
ALTER TABLE users ADD UNIQUE (email);

-- Composite unique
ALTER TABLE memberships UNIQUE (user_id, organization_id);

-- Named constraint
ALTER TABLE users ADD CONSTRAINT uq_users_email UNIQUE (email);
```

### FOREIGN KEY Constraints

```sql
CREATE TABLE posts (
    id INT PRIMARY KEY,
    author_id INT NOT NULL,
    category_id INT,
    FOREIGN KEY (author_id) REFERENCES users(id),
    FOREIGN KEY (category_id) REFERENCES categories(id)
        ON DELETE SET NULL
);

-- ON DELETE actions:
-- CASCADE: delete child rows
-- SET NULL: set FK to NULL
-- SET DEFAULT: set FK to default
-- RESTRICT: prevent delete if children exist
-- NO ACTION: same as RESTRICT (deferred check)
```

### CHECK Constraints

```sql
CREATE TABLE products (
    id INT PRIMARY KEY,
    name VARCHAR(255),
    price DECIMAL(10,2) CHECK (price >= 0),
    stock INT CHECK (stock >= 0),
    status VARCHAR(20) CHECK (status IN ('active', 'draft', 'archived'))
);
```

### NOT NULL

```sql
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    bio TEXT NULL  -- explicitly nullable
);
```

---

## 4. Upserts (Interview Focus)

### MySQL — ON DUPLICATE KEY UPDATE

```sql
-- Insert or update
INSERT INTO user_stats (user_id, login_count, last_login)
VALUES (1, 1, NOW())
ON DUPLICATE KEY UPDATE
    login_count = login_count + 1,
    last_login = NOW();

-- Works with UNIQUE constraints and PRIMARY KEY
INSERT INTO products (id, name, stock)
VALUES (1, 'Laptop', 10)
ON DUPLICATE KEY UPDATE
    stock = stock + VALUES(stock);
```

### PostgreSQL — ON CONFLICT

```sql
-- Insert or update
INSERT INTO user_stats (user_id, login_count, last_login)
VALUES (1, 1, NOW())
ON CONFLICT (user_id) DO UPDATE SET
    login_count = user_stats.login_count + 1,
    last_login = EXCLUDED.last_login;

-- Insert or do nothing
INSERT INTO products (id, name)
VALUES (1, 'Laptop')
ON CONFLICT (id) DO NOTHING;

-- With WHERE clause
INSERT INTO orders (user_id, product_id, quantity)
VALUES (1, 5, 2)
ON CONFLICT (user_id, product_id) DO UPDATE SET
    quantity = orders.quantity + EXCLUDED.quantity
WHERE orders.status = 'pending';
```

### Laravel Upserts

```php
// updateOrCreate — upsert for single record
User::updateOrCreate(
    ['email' => $email],           // match
    ['name' => $name]              // update or create
);

// insert — bulk upsert
DB::table('products')->upsert(
    [
        ['name' => 'Laptop', 'stock' => 10],
        ['name' => 'Mouse', 'stock' => 50],
    ],
    'name',  // unique column(s)
    ['stock'] // columns to update
);

// Laravel 9+ with multiple unique columns
DB::table('user_stats')->upsert(
    [
        ['user_id' => 1, 'stat' => 'logins', 'value' => 5],
        ['user_id' => 1, 'stat' => 'posts', 'value' => 3],
    ],
    ['user_id', 'stat'],  // composite unique
    ['value']              // update column
);
```

### Use Cases
- Idempotent API endpoints
- Importing CSV data
- Event counting
- Session management
- Rate limiting

---

## 5. Transactions & Locking

### Basic Transactions

```sql
-- MySQL
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;

-- On error
ROLLBACK;
```

### Laravel Transactions

```php
// Basic
DB::transaction(function () {
    Account::where('id', 1)->decrement('balance', 100);
    Account::where('id', 2)->increment('balance', 100);
});

// Nested (savepoint)
DB::transaction(function () {
    DB::transaction(function () {
        // inner transaction creates savepoint
    });
});

// Manual
DB::beginTransaction();
try {
    DB::commit();
} catch (\Exception $e) {
    DB::rollBack();
    throw $e;
}
```

### Pessimistic Locking

```sql
-- Lock row during transaction
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;

-- Lock rows matching condition
SELECT * FROM orders WHERE status = 'pending' FOR UPDATE SKIP LOCKED;

-- READ LOCK (shared — allows other reads, blocks writes)
SELECT * FROM products FOR SHARE;
```

### Optimistic Locking

```sql
-- Add version column
ALTER TABLE products ADD COLUMN version INT DEFAULT 1;

-- On update, check version hasn't changed
UPDATE products
SET name = 'Laptop', version = version + 1
WHERE id = 1 AND version = 5;

-- If affected_rows = 0, someone else modified it
```

```php
// Laravel — use package or manual
// Manual approach
$product = Product::lockForUpdate()->find(1);
$product->name = 'Laptop';
$product->version += 1;
$product->save();
```

### Deadlocks

**What causes them:**
```
Transaction A: Locks row 1, wants row 2
Transaction B: Locks row 2, wants row 1
→ Both waiting for each other → DEADLOCK
```

**Prevention:**
- Always lock rows in the same order
- Keep transactions short
- Use `SKIP LOCKED` where possible
- Use index to reduce lock scope

```sql
-- Safe: always lock in order
START TRANSACTION;
SELECT * FROM accounts WHERE id = LEAST(1, 2) FOR UPDATE;
SELECT * FROM accounts WHERE id = GREATEST(1, 2) FOR UPDATE;
```

### Isolation Levels

| Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|-------|-----------|---------------------|--------------|
| READ UNCOMMITTED | Yes | Yes | Yes |
| READ COMMITTED | No | Yes | Yes |
| REPEATABLE READ | No | No | Yes* |
| SERIALIZABLE | No | No | No |

*MySQL InnoDB: REPEATABLE READ prevents phantoms via gap locking

```sql
-- Set isolation level
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

---

## 6. Indexing

### When to Add an Index

- Columns used in WHERE clauses frequently
- Columns used in JOIN conditions
- Columns used in ORDER BY
- Columns with high cardinality (many unique values)

### Types of Indexes

```sql
-- B-tree (default, most common)
CREATE INDEX idx_users_email ON users (email);

-- Composite index (column order matters!)
CREATE INDEX idx_posts_author_status ON posts (author_id, status);
-- Can be used for: WHERE author_id = 1
-- Can be used for: WHERE author_id = 1 AND status = 'published'
-- Cannot be used for: WHERE status = 'published' (skips first column)

-- Partial index (PostgreSQL)
CREATE INDEX idx_orders_pending ON orders (user_id)
WHERE status = 'pending';

-- Covering index (includes extra columns)
CREATE INDEX idx_posts_covering ON posts (author_id, status, title);

-- Full text index
CREATE FULLTEXT INDEX idx_posts_content ON posts (content);
```

### Index Best Practices

- Don't over-index (slows writes)
- Index selectivity: index columns with high uniqueness
- Composite index column order: most selective first, or most queried first
- Don't index small tables (< 1000 rows)
- Don't index columns that are rarely queried

### EXPLAIN

```sql
-- MySQL
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';
EXPLAIN ANALYZE SELECT ...;  -- PostgreSQL: actual execution time

-- Key things to look for:
-- type: ALL (full scan — bad!) vs ref/range (good)
-- rows: estimated rows scanned
-- key: which index is used
-- Extra: "Using filesort" or "Using temporary" (bad!)
```
