SQL & PostgreSQL – In‑Depth Interview Guide (Complete Breakdown)

This guide dissects SQL and PostgreSQL from absolute fundamentals to advanced features. Every concept is explained with clear examples, internal mechanics where relevant, and practical usage. It is designed for interview preparation and deep mastery of PostgreSQL as an enterprise‑grade relational database.

---

1. Introduction to SQL and PostgreSQL

SQL (Structured Query Language) is a standard language for managing relational databases. It is declarative – you specify what you want, not how to retrieve it.

PostgreSQL is a powerful, open‑source object‑relational database system. It extends SQL with advanced features:

· ACID compliance
· MVCC (Multi‑Version Concurrency Control)
· Procedural languages (PL/pgSQL, PL/Python, PL/Perl, etc.)
· Full‑text search
· JSON/JSONB support
· Extensible data types and index methods (GiST, GIN, BRIN, SP‑GiST)
· Geographic objects (PostGIS)
· Custom aggregates, window functions
· Table inheritance, partitioning

Versions: we focus on PostgreSQL 12+ (some features may be version‑dependent).

---

2. SQL Fundamentals – The Language

SQL can be divided into sub‑languages:

· DDL (Data Definition Language): CREATE, ALTER, DROP, TRUNCATE
· DML (Data Manipulation Language): INSERT, UPDATE, DELETE, MERGE
· DQL (Data Query Language): SELECT
· DCL (Data Control Language): GRANT, REVOKE
· TCL (Transaction Control Language): COMMIT, ROLLBACK, SAVEPOINT, SET TRANSACTION

2.1 DDL – Creating and Managing Structures

CREATE TABLE

```sql
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,           -- auto-increment integer
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE,
    salary NUMERIC(10,2) CHECK (salary > 0),
    hire_date DATE DEFAULT CURRENT_DATE,
    department_id INT REFERENCES departments(id) ON DELETE SET NULL
);
```

· SERIAL / BIGSERIAL are auto‑increment shortcuts.
· PRIMARY KEY implies NOT NULL UNIQUE.
· REFERENCES defines a foreign key.
· ON DELETE can be CASCADE, SET NULL, SET DEFAULT, RESTRICT, NO ACTION.
· CHECK constraints for business rules.

ALTER TABLE

```sql
ALTER TABLE employees ADD COLUMN phone VARCHAR(20);
ALTER TABLE employees ALTER COLUMN salary SET NOT NULL;
ALTER TABLE employees DROP COLUMN phone;
ALTER TABLE employees RENAME COLUMN hire_date TO start_date;
ALTER TABLE employees ADD CONSTRAINT unique_email UNIQUE (email);
```

DROP TABLE

```sql
DROP TABLE IF EXISTS employees CASCADE; -- CASCADE drops dependent objects
```

TRUNCATE TABLE

```sql
TRUNCATE TABLE employees RESTART IDENTITY CASCADE;
-- removes all rows, resets sequences, and cascades to referencing tables.
```

2.2 DML – Data Manipulation

INSERT

```sql
INSERT INTO employees (first_name, last_name, email, salary)
VALUES ('Alice', 'Smith', 'alice@example.com', 75000);

-- multi-row
INSERT INTO employees (first_name, last_name, salary)
VALUES ('Bob', 'Jones', 60000),
       ('Carol', 'White', 80000);

-- insert from query
INSERT INTO employees_archive SELECT * FROM employees WHERE hire_date < '2020-01-01';
```

UPDATE

```sql
UPDATE employees SET salary = salary * 1.1 WHERE department_id = 2;
UPDATE employees SET email = LOWER(first_name || '.' || last_name || '@company.com')
WHERE email IS NULL;
```

DELETE

```sql
DELETE FROM employees WHERE id = 5;
DELETE FROM employees WHERE department_id IN (SELECT id FROM departments WHERE name = 'Obsolete');
```

MERGE (UPSERT) – INSERT ... ON CONFLICT

```sql
INSERT INTO employees (id, first_name, last_name, email, salary)
VALUES (1, 'Alice', 'Smith', 'alice@example.com', 80000)
ON CONFLICT (id) DO UPDATE SET salary = EXCLUDED.salary, email = EXCLUDED.email;
-- if conflict on unique constraint (id), update instead of error.
```

2.3 DQL – Querying Data

```sql
SELECT column1, column2, ...
FROM table
[WHERE condition]
[GROUP BY column]
[HAVING condition]
[ORDER BY column [ASC|DESC]]
[LIMIT count OFFSET start];
```

Basic Select

```sql
SELECT * FROM employees;
SELECT first_name, salary FROM employees WHERE department_id = 3;
```

DISTINCT

```sql
SELECT DISTINCT department_id FROM employees;
SELECT DISTINCT ON (department_id) department_id, first_name  -- PostgreSQL extension: first row per group
FROM employees ORDER BY department_id, salary DESC;
```

Aggregate Functions

· COUNT(*), COUNT(column), COUNT(DISTINCT column)
· SUM, AVG, MIN, MAX

```sql
SELECT department_id, COUNT(*) AS num_employees, AVG(salary) AS avg_salary
FROM employees
GROUP BY department_id
HAVING AVG(salary) > 50000;
```

Important: columns in SELECT must either be in GROUP BY or be aggregate functions. PostgreSQL is strict about this (only functional dependency exceptions).

Sorting and Pagination

```sql
SELECT * FROM employees ORDER BY salary DESC NULLS LAST LIMIT 10 OFFSET 20;
-- OFFSET 0 is optional. LIMIT without OFFSET returns first N rows.
```

FETCH (standard alternative to LIMIT)

```sql
SELECT * FROM employees ORDER BY salary DESC
FETCH FIRST 10 ROWS ONLY OFFSET 20 ROWS;
```

2.4 Joins

Joins combine rows from two or more tables based on a related column.

Types

· INNER JOIN – rows that match in both tables.
· LEFT [OUTER] JOIN – all rows from left, matching from right (NULL if no match).
· RIGHT [OUTER] JOIN – all rows from right, matching from left.
· FULL [OUTER] JOIN – all rows from both, NULL where no match.
· CROSS JOIN – Cartesian product.
· NATURAL JOIN – automatically joins on columns with same names (avoid in production).
· SELF JOIN – joining a table to itself.

```sql
SELECT e.first_name, d.name AS department
FROM employees e
INNER JOIN departments d ON e.department_id = d.id;

-- LEFT JOIN with null match
SELECT c.name, o.order_date
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id;

-- FULL OUTER JOIN
SELECT *
FROM table_a a
FULL JOIN table_b b ON a.key = b.key;
```

Advanced Joins

· LATERAL Join – subquery in FROM can reference columns from preceding tables.

```sql
SELECT d.name, recent_emp.first_name
FROM departments d
LEFT JOIN LATERAL (
    SELECT first_name FROM employees e
    WHERE e.department_id = d.id
    ORDER BY hire_date DESC LIMIT 1
) recent_emp ON true;
```

· USING clause – if columns have the same name: ... USING (department_id).
· Join with multiple conditions: ON a.id = b.id AND a.status = 'active'.

2.5 Subqueries

Subqueries can appear in SELECT, FROM, WHERE, HAVING.

Scalar Subquery (returns one row, one column)

```sql
SELECT first_name, salary,
    (SELECT AVG(salary) FROM employees) AS company_avg
FROM employees;
```

Row Subquery

```sql
SELECT * FROM employees
WHERE (department_id, salary) IN (
    SELECT department_id, MAX(salary) FROM employees GROUP BY department_id
);
```

Correlated Subquery (references outer query)

```sql
SELECT e.first_name, e.salary
FROM employees e
WHERE salary > (
    SELECT AVG(salary) FROM employees WHERE department_id = e.department_id
);
```

EXISTS / NOT EXISTS

```sql
SELECT * FROM departments d
WHERE EXISTS (SELECT 1 FROM employees e WHERE e.department_id = d.id);

-- Which departments have no employees?
SELECT * FROM departments d
WHERE NOT EXISTS (SELECT 1 FROM employees e WHERE e.department_id = d.id);
```

ANY / ALL

```sql
SELECT * FROM employees WHERE salary > ALL (SELECT salary FROM employees WHERE department_id = 3);
-- salary greater than every salary in dept 3.
```

2.6 Common Table Expressions (CTEs)

CTEs provide temporary result sets that can be referenced within a SELECT, INSERT, UPDATE, or DELETE statement. Use WITH.

```sql
WITH high_earners AS (
    SELECT * FROM employees WHERE salary > 70000
)
SELECT department_id, COUNT(*) FROM high_earners GROUP BY department_id;
```

Recursive CTEs – used for hierarchical data (tree traversal).

```sql
WITH RECURSIVE subordinates AS (
    -- base case: top-level manager
    SELECT id, first_name, manager_id
    FROM employees WHERE manager_id IS NULL
    UNION ALL
    -- recursive step
    SELECT e.id, e.first_name, e.manager_id
    FROM employees e
    JOIN subordinates s ON e.manager_id = s.id
)
SELECT * FROM subordinates;
```

2.7 Window Functions

Window functions perform a calculation across a set of table rows that are related to the current row. They do not collapse rows like GROUP BY.

Syntax: function_name() OVER (PARTITION BY column ORDER BY column [ROWS/RANGE frame])

Functions:

· ROW_NUMBER(), RANK(), DENSE_RANK() – ranking.
· LAG(column, offset, default), LEAD(column, offset, default) – access previous/next rows.
· FIRST_VALUE(column), LAST_VALUE(column), NTH_VALUE(column, n) – relative to frame.
· SUM(), AVG(), COUNT() as window aggregates.
· NTILE(n) – divide rows into N groups.

```sql
SELECT first_name, department_id, salary,
    RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS dept_rank,
    AVG(salary) OVER (PARTITION BY department_id) AS dept_avg,
    LAG(salary, 1) OVER (ORDER BY salary) AS prev_salary
FROM employees;
```

Frame clause (ROWS, RANGE, GROUPS): default for aggregates is RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW. You can specify ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING for moving average.

---

3. PostgreSQL Data Types

3.1 Numeric Types

· INTEGER (4 bytes), BIGINT (8), SMALLINT (2)
· SERIAL, BIGSERIAL (auto‑increment)
· DECIMAL(precision, scale), NUMERIC – exact arbitrary precision
· REAL (4 bytes), DOUBLE PRECISION (8) – inexact floating point
· MONEY – currency (with locale‑dependent formatting, better use NUMERIC)

3.2 Character Types

· CHAR(n) / CHARACTER(n) – fixed‑length, blank‑padded.
· VARCHAR(n) / CHARACTER VARYING(n) – variable‑length with limit.
· TEXT – unlimited length (preferred in PostgreSQL; no performance penalty).

3.3 Date/Time Types

· DATE – date only.
· TIME / TIME WITH TIME ZONE – time of day.
· TIMESTAMP / TIMESTAMPTZ (timestamp with time zone) – date + time.
· INTERVAL – span of time (INTERVAL '1 day 3 hours').
· CURRENT_DATE, CURRENT_TIME, CURRENT_TIMESTAMP.

```sql
SELECT TIMESTAMP '2024-01-15 14:30:00' AT TIME ZONE 'UTC';
```

3.4 Boolean

BOOLEAN – TRUE, FALSE, NULL. Can use 't', 'true', 'yes', 'on', '1' etc.

3.5 Arrays

PostgreSQL supports arrays of any built‑in or user‑defined type.

```sql
CREATE TABLE sal_emp (
    name TEXT,
    pay_by_quarter INTEGER[],
    schedule TEXT[][]
);
INSERT INTO sal_emp VALUES ('Bill', '{10000, 10000, 10000, 10000}', '{{"meeting", "lunch"}, {"training", "presentation"}}');

SELECT pay_by_quarter[1] FROM sal_emp;
SELECT * FROM sal_emp WHERE 10000 = ANY(pay_by_quarter);
SELECT * FROM sal_emp WHERE pay_by_quarter @> '{10000,10000}';  -- contains
```

Array operators and functions: @>, <@, && (overlap), || (concatenation), array_append, array_remove, array_length, unnest() to expand into rows.

3.6 JSON/JSONB

PostgreSQL provides two JSON data types:

· JSON – stores exact copy of input text, preserves whitespace/ordering, slower processing (re‑parsed each time).
· JSONB – binary representation, supports indexing, faster processing, does not preserve whitespace/order of object keys.

Use JSONB for most use cases.

```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    data JSONB
);

INSERT INTO products (data) VALUES ('{"name": "Laptop", "price": 1200, "tags": ["electronics", "portable"]}');

-- Querying JSON
SELECT data -> 'name' AS name FROM products;  -- returns JSON
SELECT data ->> 'name' AS name FROM products; -- returns text
SELECT * FROM products WHERE data @> '{"price": 1200}';  -- contains operator
SELECT * FROM products WHERE data ? 'tags';  -- existence of key
SELECT jsonb_array_elements(data->'tags') FROM products;
```

Indexes on JSONB:

· GIN index: CREATE INDEX idxgin ON products USING GIN (data);
· GIN index on specific path: CREATE INDEX idx_name ON products USING GIN ((data -> 'name'));
· For @> operator, GIN is efficient.
· B‑tree on expressions: CREATE INDEX idx_price ON products ((data->>'price')::numeric);

3.7 Other Data Types

· UUID – Universally Unique Identifier.
· BYTEA – binary data.
· NETWORK ADDRESS: INET, CIDR, MACADDR.
· GEOMETRIC: POINT, LINE, LSEG, BOX, PATH, POLYGON, CIRCLE.
· RANGE types: INT4RANGE, INT8RANGE, NUMRANGE, TSRANGE (timestamp range), TSTZRANGE, DATERANGE. Range operators like @> (contains), && (overlaps).
· HSTORE – key‑value store within a single value (similar to JSON but older; mostly superseded by JSONB).
· XML – XML data.
· PG_LSN – Log Sequence Number.
· TXID_SNAPSHOT – snapshot for MVCC.

3.8 Domains

Domain is a user‑defined data type with constraints.

```sql
CREATE DOMAIN positive_numeric AS NUMERIC CHECK (VALUE > 0);
CREATE TABLE items (
    id SERIAL,
    price positive_numeric
);
```

3.9 Enumerated Types

```sql
CREATE TYPE mood AS ENUM ('happy', 'sad', 'neutral');
CREATE TABLE person (
    name TEXT,
    current_mood mood
);
```

Enum values are stored as integers internally.

---

4. Constraints

Constraints enforce data integrity.

· NOT NULL – column cannot have NULL.
· UNIQUE – all values distinct (NULLs are considered distinct in PostgreSQL; multiple NULLs allowed).
· PRIMARY KEY – UNIQUE + NOT NULL. Only one per table.
· CHECK – boolean expression, e.g., salary > 0.
· FOREIGN KEY – references primary key in another table. Ensures referential integrity.
· EXCLUSION – PostgreSQL extension; uses GiST to enforce constraints like non‑overlapping ranges.

```sql
-- Exclusion constraint: no overlapping room bookings
CREATE TABLE reservations (
    room_id INT,
    during TSRANGE,
    EXCLUDE USING GIST (room_id WITH =, during WITH &&)
);
```

Adding/removing constraints:

```sql
ALTER TABLE employees ADD CONSTRAINT fk_dept FOREIGN KEY (department_id) REFERENCES departments(id);
ALTER TABLE employees DROP CONSTRAINT fk_dept;
```

Deferrable constraints: constraints can be checked at commit time rather than immediately, useful for cyclic foreign keys.

```sql
ALTER TABLE employees ADD CONSTRAINT fk_dept FOREIGN KEY (dept_id) REFERENCES depts(id) DEFERRABLE INITIALLY DEFERRED;
```

---

5. Indexes

Indexes speed up data retrieval.

5.1 Types of Indexes

· B‑tree (default) – good for equality and range queries on sortable data.
· Hash – only equality comparisons.
· GiST (Generalized Search Tree) – for full‑text search, geometric data, range types.
· GIN (Generalized Inverted Index) – for arrays, JSONB, full‑text search (handles many keys).
· SP‑GiST – space‑partitioned GiST; for non‑balanced data structures.
· BRIN (Block Range Index) – for very large tables with natural ordering (e.g., time series). Very small.
· Bloom – for many columns combined.

5.2 Creating Indexes

```sql
CREATE INDEX idx_emp_name ON employees (last_name);
CREATE INDEX idx_emp_dept_salary ON employees (department_id, salary);  -- composite index
CREATE UNIQUE INDEX unique_lower_email ON employees (LOWER(email));  -- expression index
CREATE INDEX idx_emp_gin ON employees USING GIN (some_jsonb);
CREATE INDEX idx_emp_gist ON employees USING GIST (location);
-- Partial index
CREATE INDEX idx_active_emp ON employees (salary) WHERE status = 'active';
-- Index with included columns (PostgreSQL 11+)
CREATE INDEX idx_emp_covering ON employees (last_name) INCLUDE (first_name, salary);
```

5.3 Index Mechanics

· B‑tree fills leaf pages and maintains balance. fillfactor can be set to leave room for updates.
· REINDEX rebuilds an index.
· CLUSTER orders the table physically according to an index (one‑time, not maintained).
· VACUUM cleans up dead tuples, updates index statistics.
· Index only scans: if all required columns are in the index (including INCLUDE), PostgreSQL can avoid reading the table (visibility check via VACUUM is needed).

5.4 Monitoring

· EXPLAIN ANALYZE shows index usage.
· pg_stat_user_indexes and pg_stat_all_indexes for statistics.
· Use pg_stat_reset() to reset counters.

5.5 When Indexes Are Not Used

· Function on column without matching expression index.
· Leading wildcard LIKE '%abc'.
· Very small tables (sequential scan faster).
· Poor statistics – ANALYZE to update.

---

6. Views and Materialized Views

6.1 Views

A view is a saved query. It does not store data; each access re‑evaluates the query.

```sql
CREATE VIEW high_salary_employees AS
SELECT first_name, last_name, salary FROM employees WHERE salary > 70000;

SELECT * FROM high_salary_employees WHERE salary > 80000;
```

· Simple views can be updatable (if they don't contain aggregates, joins, DISTINCT, etc.). Use CREATE RULE or INSTEAD OF trigger for complex views.
· WITH CHECK OPTION ensures inserted/updated rows satisfy the view condition.

6.2 Materialized Views

Stores the result set physically. Faster reads, but data becomes stale until refreshed.

```sql
CREATE MATERIALIZED VIEW sales_summary AS
SELECT department_id, SUM(amount) AS total_sales
FROM sales GROUP BY department_id;

REFRESH MATERIALIZED VIEW sales_summary;            -- recompute completely (blocks reads)
REFRESH MATERIALIZED VIEW CONCURRENTLY sales_summary; -- allows reads, requires unique index
```

· Concurrent refresh needs at least one unique index on the materialized view.

---

7. Transactions, Concurrency, and Isolation

7.1 Transaction Basics

```sql
BEGIN; -- or START TRANSACTION
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT; -- or ROLLBACK;
```

· SAVEPOINT for partial rollback: SAVEPOINT my_savepoint; ROLLBACK TO my_savepoint;
· SET TRANSACTION to set isolation level, read only, etc.

7.2 ACID Properties

· Atomicity: all or nothing.
· Consistency: database moves from one valid state to another.
· Isolation: concurrent transactions do not interfere (level dependent).
· Durability: committed data survives crashes (WAL – Write‑Ahead Log).

7.3 Isolation Levels

Defined by the SQL standard. PostgreSQL implements:

· READ UNCOMMITTED – identical to READ COMMITTED in PostgreSQL (no dirty reads allowed).
· READ COMMITTED (default) – a query sees data committed before it began; may see changes committed during transaction (non‑repeatable reads).
· REPEATABLE READ – a query sees only data committed before the transaction began; no non‑repeatable reads. Phantom reads still possible.
· SERIALIZABLE – strictest; transactions appear to run sequentially; uses SSI (Serializable Snapshot Isolation). May fail with serialization error – need to retry.

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
-- queries
COMMIT;
```

7.4 MVCC – Multi‑Version Concurrency Control

PostgreSQL uses MVCC to avoid locking for reads. Each row has multiple versions; a transaction sees a snapshot of the database at a particular LSN. No read locks; writes never block reads (except exclusive locks). This is why VACUUM is needed to clean up old row versions.

7.5 Locks

PostgreSQL has table‑level and row‑level locks.

Table‑level locks (automatically acquired or via LOCK TABLE):

· ACCESS SHARE (SELECT) – conflicts only with ACCESS EXCLUSIVE.
· ROW SHARE (SELECT FOR UPDATE/SHARE) – conflicts with EXCLUSIVE and ACCESS EXCLUSIVE.
· ROW EXCLUSIVE (INSERT/UPDATE/DELETE) – conflicts with SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE.
· SHARE (CREATE INDEX CONCURRENTLY) – etc.
· SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE (ALTER TABLE, DROP TABLE) – highest.

Row‑level locks:

· FOR UPDATE – exclusive lock on selected rows (other transactions cannot FOR UPDATE or FOR SHARE the same rows until commit).
· FOR SHARE – shared lock, allows other FOR SHARE but blocks FOR UPDATE.
· FOR NO KEY UPDATE (PostgreSQL 9.3+) – weaker lock, blocks FOR KEY SHARE but not simple updates.

```sql
BEGIN;
SELECT * FROM orders WHERE id = 5 FOR UPDATE;
-- now row locked
COMMIT; -- release
```

7.6 Deadlocks

Occur when two transactions wait on each other's locks. PostgreSQL detects deadlocks and aborts one transaction.

---

8. Advanced Querying

8.1 Set Operations

· UNION [ALL] – combine rows from two queries (ALL includes duplicates).
· INTERSECT [ALL]
· EXCEPT [ALL]

```sql
SELECT name FROM employees
UNION
SELECT name FROM contractors;
```

8.2 CASE Expression

```sql
SELECT first_name,
    CASE
        WHEN salary > 80000 THEN 'High'
        WHEN salary > 50000 THEN 'Medium'
        ELSE 'Low'
    END AS salary_level
FROM employees;
```

8.3 COALESCE and NULLIF

· COALESCE(value1, value2, ...) returns first non‑null argument.
· NULLIF(expr1, expr2) returns NULL if equal, else expr1.

8.4 GREATEST and LEAST

Return the largest/smallest value from a list.

8.5 STRING_AGG (Aggregate String Concatenation)

```sql
SELECT department_id, STRING_AGG(first_name, ', ' ORDER BY first_name) AS employees
FROM employees GROUP BY department_id;
```

8.6 ARRAY_AGG

```sql
SELECT department_id, ARRAY_AGG(first_name ORDER BY hire_date) AS emp_names
FROM employees GROUP BY department_id;
```

8.7 FILTER Clause for Aggregate

```sql
SELECT department_id,
    COUNT(*) FILTER (WHERE salary > 50000) AS high_earners,
    COUNT(*) AS total
FROM employees GROUP BY department_id;
```

8.8 DISTINCT ON (PostgreSQL extension)

```sql
SELECT DISTINCT ON (department_id) department_id, first_name, salary
FROM employees
ORDER BY department_id, salary DESC;
-- first row per department by salary
```

8.9 CROSSTAB (Pivot Tables via tablefunc extension)

Requires CREATE EXTENSION tablefunc;. Not covered in detail but useful.

---

9. Functions, Procedures, and PL/pgSQL

9.1 Functions

· Return a value (scalar, table, set). Created with CREATE FUNCTION.
· Can be written in SQL, PL/pgSQL, C, etc.
· IMMUTABLE, STABLE, VOLATILE categories for optimizer.
· RETURNS TABLE, RETURNS SETOF.

```sql
CREATE FUNCTION get_employees_by_dept(dept_id INT)
RETURNS TABLE(id INT, name TEXT, salary NUMERIC) AS $$
BEGIN
    RETURN QUERY SELECT e.id, e.first_name, e.salary
    FROM employees e WHERE e.department_id = dept_id;
END;
$$ LANGUAGE plpgsql;
```

9.2 Stored Procedures (PostgreSQL 11+)

Support transaction control (COMMIT, ROLLBACK) inside the body. Created with CREATE PROCEDURE. Use CALL to invoke.

```sql
CREATE PROCEDURE transfer_funds(sender INT, receiver INT, amount NUMERIC)
LANGUAGE plpgsql AS $$
BEGIN
    UPDATE accounts SET balance = balance - amount WHERE id = sender;
    UPDATE accounts SET balance = balance + amount WHERE id = receiver;
    COMMIT;
END;
$$;

CALL transfer_funds(1, 2, 100);
```

9.3 Triggers

Procedural code executed automatically in response to events (INSERT, UPDATE, DELETE). Can be BEFORE, AFTER, INSTEAD OF.

```sql
CREATE FUNCTION log_employee_changes() RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO employee_audit(employee_id, changed_on, action)
    VALUES (NEW.id, NOW(), TG_OP);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER emp_audit
AFTER INSERT OR UPDATE OR DELETE ON employees
FOR EACH ROW EXECUTE FUNCTION log_employee_changes();
```

· OLD and NEW record variables.
· TG_OP contains operation name.
· FOR EACH ROW vs FOR EACH STATEMENT.
· INSTEAD OF triggers on views to make them updatable.

9.4 Event Triggers (PostgreSQL 9.3+)

Trigger on DDL commands (CREATE TABLE, ALTER, etc.) for database‑level auditing.

---

10. Full‑Text Search

PostgreSQL has built‑in full‑text search via tsvector and tsquery.

```sql
-- to_tsvector converts text to searchable tokens
SELECT to_tsvector('english', 'The quick brown fox jumped over the lazy dog') @@ to_tsquery('fox & dog'); -- true

-- Create a column for precomputed tsvector
ALTER TABLE articles ADD COLUMN tsv tsvector;
UPDATE articles SET tsv = to_tsvector('english', title || ' ' || body);
CREATE INDEX idx_fts ON articles USING GIN (tsv);

-- Query
SELECT * FROM articles WHERE tsv @@ to_tsquery('english', 'database & optimization');
```

· Ranking functions: ts_rank, ts_rank_cd.
· Highlighting: ts_headline.
· Dictionaries, stop words, thesaurus.

---

11. Partitioning

Divide large tables into smaller physical pieces while presenting a single logical table. PostgreSQL supports:

· Range Partitioning – by key value ranges.
· List Partitioning – by discrete values.
· Hash Partitioning – by hash.

Declarative partitioning (PostgreSQL 10+):

```sql
CREATE TABLE sales (
    id SERIAL,
    sale_date DATE NOT NULL,
    amount NUMERIC
) PARTITION BY RANGE (sale_date);

CREATE TABLE sales_2024_q1 PARTITION OF sales
FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');

CREATE TABLE sales_2024_q2 PARTITION OF sales
FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');
```

· Sub‑partitioning possible.
· PRIMARY KEY must include the partition key.
· UPDATE that moves a row to a different partition is not allowed (must delete+insert).

Partition Pruning

Planner can skip scanning partitions that cannot contain relevant rows.

---

12. Replication and High Availability

12.1 Streaming Replication

Master‑standby(s). WAL records streamed to standbys. Asynchronous (default) or synchronous (synchronous_commit = remote_apply).

12.2 Logical Replication

Row‑level replication, allowing selective table replication and cross‑version upgrades. Uses publications and subscriptions.

12.3 Failover

Tools like Patroni, repmgr, pgpool‑II for automatic failover.

---

13. Backup and Restore

· pg_dump: logical backup of a single database.
  ```bash
  pg_dump -U username -d dbname > backup.sql
  psql -d dbname -f backup.sql
  ```
· pg_dumpall: backup all databases and global objects.
· pg_basebackup: physical base backup for point‑in‑time recovery.
· Point‑in‑time recovery (PITR): combine base backup with WAL archives.

---

14. Extensions

PostgreSQL is extensible. Popular extensions:

· uuid-ossp – generate UUIDs.
· pg_trgm – trigram fuzzy string matching (LIKE '%abc%' index).
· hstore – key‑value store.
· postgis – geospatial data.
· pg_stat_statements – query performance analysis.
· pgcrypto – encryption functions.
· tablefunc – crosstab.
· ltree – tree structures.
· pg_partman – automatic partition management.

Enable: CREATE EXTENSION IF NOT EXISTS pg_trgm;

---

15. Performance Tuning and EXPLAIN

15.1 EXPLAIN and EXPLAIN ANALYZE

```sql
EXPLAIN SELECT * FROM employees WHERE salary > 50000;
EXPLAIN (ANALYZE, BUFFERS, VERBOSE) SELECT ...;
```

Analyze shows actual times, row counts, loops. Buffers shows cache usage.

Nodes: Seq Scan, Index Scan, Index Only Scan, Bitmap Heap Scan, Nested Loop, Hash Join, Merge Join.

Key metrics:

· cost=start..end – in arbitrary units (page fetches). start is startup cost, end is total.
· rows – estimated number of rows.
· width – average row width.
· actual time=... loops=... (with ANALYZE).

Optimization tips:

· High startup cost may mean sort.
· Hash join for large unsorted inputs; merge join if both sorted.
· Sequential scan may be better than index for large fraction.

15.2 Configuration Parameters

· shared_buffers – memory for caching data. Typically 25% of system RAM.
· work_mem – memory for sorts/hash operations per operation (can be set per query). Increase for large aggregations.
· maintenance_work_mem – for VACUUM, CREATE INDEX.
· effective_cache_size – hints optimizer about OS cache.
· random_page_cost – lower for SSDs (1.1 vs 4 for HDD).
· seq_page_cost, cpu_tuple_cost, etc.

15.3 ANALYZE and VACUUM

· ANALYZE updates table statistics (column histograms, most common values). Run after bulk changes.
· VACUUM reclaims storage from dead tuples. Autovacuum daemon runs automatically.
· VACUUM FULL rewrites the entire table (locks exclusively, reclaims disk space).

---

16. Security and User Management

16.1 Roles

A role can be a user or a group. CREATE ROLE, CREATE USER (same with login privilege).

```sql
CREATE ROLE app_user LOGIN PASSWORD 'secret';
GRANT SELECT, INSERT ON employees TO app_user;
REVOKE DELETE ON employees FROM app_user;
```

· SUPERUSER, CREATEDB, CREATEROLE attributes.
· GRANT role1 TO role2 – role2 inherits privileges of role1 (unless NOINHERIT).

16.2 Row‑Level Security (RLS)

Restrict which rows a user can access based on policy.

```sql
ALTER TABLE employees ENABLE ROW LEVEL SECURITY;
CREATE POLICY emp_dept_policy ON employees
    USING (department_id IN (SELECT id FROM departments WHERE manager = current_user));
```

· USING for rows visible; WITH CHECK for rows inserted/updated.

16.3 Column‑Level Privileges

GRANT SELECT (col1, col2) ON table TO role;

16.4 SSL Encryption

Configure ssl = on in postgresql.conf, provide certificates. Clients can connect with sslmode=require.

---

17. Common Extensions and Advanced Features

17.1 pg_stat_statements

Records query statistics. Useful to find slow/frequent queries.

```sql
CREATE EXTENSION pg_stat_statements;
SELECT query, calls, total_time, mean_time
FROM pg_stat_statements ORDER BY total_time DESC;
```

17.2 pg_trgm – Trigram Fuzzy Matching

```sql
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_trgm ON employees USING GIN (first_name gin_trgm_ops);
SELECT * FROM employees WHERE first_name % 'Jon';  -- similarity > threshold
SELECT similarity('John', 'Jon');  -- 0.6
```

17.3 btree_gist

Allows GiST indexes with B‑tree operators, needed for exclusion constraints involving scalars.

17.4 pgcrypto

Provides cryptographic functions: digest(), crypt(), gen_random_uuid(), encrypt(), decrypt().

---

18. Database Design and Normalization

· 1NF: atomic values, no repeating groups.
· 2NF: 1NF + no partial dependency (non‑prime attributes depend on whole primary key).
· 3NF: 2NF + no transitive dependency.
· BCNF, 4NF, 5NF: higher levels.

Normalization reduces redundancy. Denormalization may be needed for performance.

PostgreSQL features for design: table inheritance, composite types, arrays (but careful with denormalization).

---

19. Common Interview Questions and Answers

1. Explain ACID properties.
2. What is MVCC and how does PostgreSQL implement it?
3. Difference between DELETE and TRUNCATE? (DELETE logged per row, can rollback, TRUNCATE fast, not logged per row, resets sequences, cannot be used with WHERE.)
4. What are window functions? Give an example.
5. How to find duplicates in a table? SELECT email, COUNT(*) FROM users GROUP BY email HAVING COUNT(*) > 1;
6. What is a correlated subquery? How does it differ from a simple subquery?
7. Explain the different types of JOINs.
8. What is an index? Types in PostgreSQL?
9. What is a materialized view? When would you use it?
10. Explain the difference between UNION and UNION ALL.
11. How does LIMIT/OFFSET work? What are the performance issues with large offsets? (Often materialize and skip.)
12. What is a deadlock? How to detect and prevent?
13. What is the role of VACUUM in PostgreSQL?
14. Explain EXPLAIN ANALYZE output.
15. What is a recursive CTE? Provide an example.
16. How to handle JSON in PostgreSQL? Differences between JSON and JSONB?
17. What is a LATERAL join?
18. Explain isolation levels and phantom reads.
19. How to implement pagination efficiently? (Keyset pagination: WHERE id > last_id ORDER BY id LIMIT n.)
20. What are the advantages of PostgreSQL over other RDBMS?

---

This guide covers SQL and PostgreSQL from the basics of table creation to advanced features like window functions, JSONB, full‑text search, and replication. Use it as a comprehensive reference for interviews and deep database engineering.
