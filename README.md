# PostgreSQL: A Practical Reference

A structured reference for learning and using PostgreSQL (written for PostgreSQL 16). It moves from
core concepts to daily querying, then to transactions, performance, security and operations.

## Contents

1. [SQL and PostgreSQL](#1-sql-and-postgresql)
2. [Core concepts](#2-core-concepts)
3. [Data types](#3-data-types)
4. [Defining tables and constraints](#4-defining-tables-and-constraints)
5. [Querying data](#5-querying-data)
6. [Joins and aggregation](#6-joins-and-aggregation)
7. [Advanced queries](#7-advanced-queries)
8. [Transactions, MVCC and locking](#8-transactions-mvcc-and-locking)
9. [Indexes and query plans](#9-indexes-and-query-plans)
10. [Programmability](#10-programmability)
11. [Roles, privileges and row-level security](#11-roles-privileges-and-row-level-security)
12. [Operations](#12-operations)
13. [Common mistakes](#13-common-mistakes)
14. [Learning order](#14-learning-order)
15. [Practice questions](#15-practice-questions)
16. [References](#16-references)

---

## 1. SQL and PostgreSQL

**SQL** is a standardised language (ISO/IEC 9075) for defining and querying relational data.
**PostgreSQL** is a database management system (DBMS) that implements SQL and adds its own extensions.

| Aspect | SQL | PostgreSQL |
|---|---|---|
| Nature | Language specification | Server software (a DBMS) |
| Provides | Syntax and semantics for queries | Storage engine, planner, transactions, replication |
| Portability | Largely portable | Extensions (`jsonb`, arrays, `RETURNING`) are PostgreSQL-specific |

Other systems implementing SQL include MySQL, SQLite, Oracle and SQL Server. Each differs in dialect and features.

## 2. Core concepts

- **Relation (table):** a set of rows sharing the same columns (Codd, 1970).
- **Row (tuple) and column (attribute):** a record and one of its typed fields.
- **Schema:** a namespace inside a database (default: `public`).
- **Primary key:** uniquely identifies a row. **Foreign key:** references a key in another table.
- **Transaction:** a group of statements that succeed or fail together (ACID).

```text
cluster ─▶ database ─▶ schema ─▶ table / view / index / function
```

Connecting and inspecting with `psql`:

```bash
psql -h localhost -U app_user -d appdb
```

```text
\l            list databases          \dt           list tables
\d tablename  describe a table        \du           list roles
\x            toggle expanded output  \timing       show query time
```

## 3. Data types

| Category | Common types | Notes |
|---|---|---|
| Integers | `smallint`, `integer`, `bigint` | Prefer `bigint` for identifiers |
| Exact numbers | `numeric(p,s)` | Use for money; never `float` |
| Text | `text`, `varchar(n)` | `text` has no performance penalty versus `varchar` |
| Boolean | `boolean` | `true`, `false`, `NULL` |
| Date and time | `date`, `timestamptz`, `interval` | Prefer `timestamptz` over `timestamp` |
| Identifiers | `uuid` | `gen_random_uuid()` is built in |
| Semi-structured | `jsonb`, arrays | `jsonb` is binary and indexable |
| Enumerations | `CREATE TYPE ... AS ENUM` | Adding values is easy; removing is not |

`NULL` means "unknown". Comparisons with `NULL` yield `NULL`, so use `IS NULL` and `IS DISTINCT FROM`.

## 4. Defining tables and constraints

```sql
CREATE TABLE customers (
    id         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email      text        NOT NULL UNIQUE,
    name       text        NOT NULL,
    created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE orders (
    id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id bigint      NOT NULL REFERENCES customers(id) ON DELETE RESTRICT,
    status      text        NOT NULL DEFAULT 'new'
                CHECK (status IN ('new','paid','shipped','cancelled')),
    placed_at   timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE order_items (
    order_id   bigint  NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    line_no    integer NOT NULL,
    sku        text    NOT NULL,
    quantity   integer NOT NULL CHECK (quantity > 0),
    unit_price numeric(12,2) NOT NULL CHECK (unit_price >= 0),
    PRIMARY KEY (order_id, line_no)          -- composite key
);
```

- `GENERATED ... AS IDENTITY` is the standard replacement for `serial`.
- Foreign keys do **not** create an index on the referencing column; add one if you join or delete through it.
- `ON DELETE` actions: `RESTRICT`, `NO ACTION`, `CASCADE`, `SET NULL`, `SET DEFAULT`.

Changing a schema:

```sql
ALTER TABLE customers ADD COLUMN phone text;
ALTER TABLE customers ALTER COLUMN name SET NOT NULL;
ALTER TABLE customers DROP COLUMN phone;
```

## 5. Querying data

```sql
INSERT INTO customers (email, name)
VALUES ('ada@example.org', 'Ada')
RETURNING id;

UPDATE orders SET status = 'paid' WHERE id = 1 RETURNING *;
DELETE FROM orders WHERE status = 'cancelled';

SELECT id, email
FROM   customers
WHERE  created_at >= now() - interval '30 days'
ORDER  BY created_at DESC
LIMIT  10 OFFSET 0;
```

Logical evaluation order: `FROM` → `WHERE` → `GROUP BY` → `HAVING` → `SELECT` → `ORDER BY` → `LIMIT`.

Upsert:

```sql
INSERT INTO customers (email, name)
VALUES ('ada@example.org', 'Ada L.')
ON CONFLICT (email) DO UPDATE SET name = EXCLUDED.name;
```

Pagination: `OFFSET` scans and discards rows. For large tables use keyset pagination:

```sql
SELECT * FROM orders WHERE id > 1000 ORDER BY id LIMIT 50;
```

## 6. Joins and aggregation

```sql
-- INNER JOIN: only matching rows
SELECT c.name, o.id
FROM   customers c JOIN orders o ON o.customer_id = c.id;

-- LEFT JOIN: keep all customers, even without orders
SELECT c.name, count(o.id) AS orders
FROM   customers c LEFT JOIN orders o ON o.customer_id = c.id
GROUP  BY c.id, c.name
HAVING count(o.id) >= 0
ORDER  BY orders DESC;
```

| Join | Returns |
|---|---|
| `INNER` | Rows matching in both tables |
| `LEFT` / `RIGHT` | All rows from one side, matches from the other (else `NULL`) |
| `FULL` | All rows from both sides |
| `CROSS` | Cartesian product |

`WHERE` filters rows before grouping; `HAVING` filters groups after aggregation.

Aggregates: `count`, `sum`, `avg`, `min`, `max`, `string_agg`, `array_agg`, and `FILTER`:

```sql
SELECT count(*) FILTER (WHERE status = 'paid') AS paid,
       count(*)                                AS total
FROM   orders;
```

## 7. Advanced queries

**Subqueries and `EXISTS`**

```sql
SELECT * FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```

**Common table expressions (CTEs)**

```sql
WITH recent AS (
    SELECT * FROM orders WHERE placed_at > now() - interval '7 days'
)
SELECT customer_id, count(*) FROM recent GROUP BY customer_id;
```

**Recursive CTEs** (hierarchies)

```sql
WITH RECURSIVE n(i) AS (
    SELECT 1
    UNION ALL
    SELECT i + 1 FROM n WHERE i < 5
)
SELECT * FROM n;
```

**Window functions**

```sql
SELECT id, customer_id, placed_at,
       row_number() OVER (PARTITION BY customer_id ORDER BY placed_at DESC) AS rn,
       sum(1)       OVER (PARTITION BY customer_id)                          AS customer_orders
FROM   orders;
```

**`jsonb`**

```sql
CREATE TABLE events (id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY, payload jsonb NOT NULL);
INSERT INTO events (payload) VALUES ('{"type":"login","user":{"id":7}}');

SELECT payload->>'type', payload->'user'->>'id' FROM events;
SELECT * FROM events WHERE payload @> '{"type":"login"}';
CREATE INDEX events_payload_gin ON events USING gin (payload);
```

**Full-text search**

```sql
CREATE TABLE docs (id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY, body text,
    tsv tsvector GENERATED ALWAYS AS (to_tsvector('english', body)) STORED);
CREATE INDEX docs_tsv ON docs USING gin (tsv);
SELECT id FROM docs WHERE tsv @@ websearch_to_tsquery('english', 'postgres index');
```

**Set operations:** `UNION`, `INTERSECT`, `EXCEPT` (add `ALL` to keep duplicates).

## 8. Transactions, MVCC and locking

A transaction groups statements; PostgreSQL guarantees **ACID**: atomicity, consistency, isolation, durability.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;            -- or ROLLBACK;
```

**MVCC (multiversion concurrency control).** An update creates a new row version instead of overwriting.
Readers see a snapshot and do not block writers, and writers do not block readers. Old versions are removed by
`VACUUM`.

**Isolation levels**

| Level | Dirty read | Non-repeatable read | Phantom | Notes |
|---|---|---|---|---|
| Read Committed (default) | No | Possible | Possible | New snapshot per statement |
| Repeatable Read | No | No | No in PostgreSQL | Snapshot per transaction; may raise serialization failures |
| Serializable | No | No | No | Serializable Snapshot Isolation; retry on error `40001` |

PostgreSQL treats `READ UNCOMMITTED` as `READ COMMITTED`. Background: Berenson et al., 1995.

**Explicit row locks**

```sql
SELECT * FROM orders WHERE id = 1 FOR UPDATE;           -- exclusive row lock
SELECT * FROM orders WHERE id = 1 FOR SHARE;            -- block writers, allow readers

-- Work queue: each worker takes a different row without waiting
SELECT id FROM jobs WHERE status = 'queued'
ORDER BY id LIMIT 1
FOR UPDATE SKIP LOCKED;
```

**Deadlocks** occur when transactions wait for each other in a cycle. PostgreSQL aborts one. Avoid them by
locking rows in a consistent order.

**Advisory locks:** `pg_advisory_xact_lock(key)` provides application-level mutual exclusion.

## 9. Indexes and query plans

| Index | Use for |
|---|---|
| B-tree (default) | Equality and range, sorting |
| Hash | Equality only |
| GIN | `jsonb`, arrays, full-text search |
| GiST | Geometry, ranges, nearest-neighbour |
| BRIN | Very large, naturally ordered tables |

```sql
CREATE INDEX orders_customer_idx ON orders (customer_id);
CREATE INDEX orders_paid_idx ON orders (placed_at) WHERE status = 'paid';   -- partial
CREATE UNIQUE INDEX customers_lower_email ON customers (lower(email));       -- expression
CREATE INDEX CONCURRENTLY orders_status_idx ON orders (status);             -- no write lock
```

Reading a plan:

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE customer_id = 42;
```

| Plan node | Meaning |
|---|---|
| `Seq Scan` | Reads the whole table |
| `Index Scan` / `Index Only Scan` | Uses an index (and possibly the heap) |
| `Bitmap Heap Scan` | Collects matching pages, then reads them |
| `Nested Loop` / `Hash Join` / `Merge Join` | Join strategies |

Guidelines: index columns used in `WHERE`, `JOIN` and `ORDER BY`; in a multi-column index put equality columns
first; every index slows writes; run `ANALYZE` after large changes so statistics stay current.

## 10. Programmability

```sql
-- View
CREATE VIEW paid_orders AS SELECT * FROM orders WHERE status = 'paid';

-- Function
CREATE FUNCTION order_total(p_order bigint) RETURNS numeric
LANGUAGE sql STABLE AS $$
    SELECT coalesce(sum(quantity * unit_price), 0) FROM order_items WHERE order_id = p_order;
$$;

-- Trigger
CREATE FUNCTION touch_updated() RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN NEW.updated_at := now(); RETURN NEW; END $$;

CREATE TRIGGER trg_touch BEFORE UPDATE ON some_table
FOR EACH ROW EXECUTE FUNCTION touch_updated();
```

Also available: materialized views (`REFRESH MATERIALIZED VIEW`), stored procedures (`CALL`), `LISTEN`/`NOTIFY`,
table partitioning, and extensions (`CREATE EXTENSION`, for example `pg_trgm`, `pgcrypto`, `pgvector`).

## 11. Roles, privileges and row-level security

```sql
CREATE ROLE app_user LOGIN PASSWORD '<secret>';
CREATE ROLE readonly NOLOGIN;
GRANT CONNECT ON DATABASE appdb TO app_user;
GRANT USAGE ON SCHEMA public TO readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;
GRANT readonly TO app_user;
```

**Row-level security**

```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
CREATE POLICY own_orders ON orders
    USING (customer_id = current_setting('app.customer_id')::bigint);
```

Security practice:

- Use parameterised queries; never build SQL by string concatenation (SQL injection).
- Give applications a role with the minimum privileges, not a superuser.
- Require TLS (`sslmode=verify-full`) for remote connections.
- Store password hashes with `scram-sha-256` (`password_encryption`).
- Restrict network access in `pg_hba.conf`:

```text
# TYPE  DATABASE  USER      ADDRESS        METHOD
hostssl appdb     app_user  10.0.0.0/24    scram-sha-256
```

## 12. Operations

**Backup and restore**

```bash
pg_dump  -Fc -d appdb -f appdb.dump                  # logical backup, custom format
pg_restore -d appdb_new --no-owner appdb.dump        # restore
pg_dumpall --globals-only > globals.sql              # roles and tablespaces
```

For point-in-time recovery (PITR), take a base backup (`pg_basebackup`) and archive WAL files
(`archive_mode = on`, `archive_command`).

**Maintenance**

- `VACUUM` reclaims dead row versions; `autovacuum` runs it automatically. `VACUUM FULL` rewrites the table and locks it.
- `ANALYZE` refreshes planner statistics.
- `REINDEX` rebuilds bloated indexes.

**Monitoring**

```sql
SELECT pid, state, wait_event, query FROM pg_stat_activity WHERE state <> 'idle';
SELECT relname, n_dead_tup FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 10;
```

**Connections:** each connection is a server process, so use a pool (for example PgBouncer) when there are many clients.

**Replication:** streaming (physical) replication for standbys and failover; logical replication for selected tables.

**Migrations:** apply schema changes as ordered, versioned scripts, in a transaction where possible.
PostgreSQL supports transactional DDL.

## 13. Common mistakes

| Mistake | Better practice |
|---|---|
| `timestamp` without time zone | Use `timestamptz` |
| `serial` for new tables | Use `GENERATED ... AS IDENTITY` |
| `float` for money | Use `numeric` |
| `SELECT *` in application code | List the columns you need |
| `= NULL` | `IS NULL` |
| No index on a foreign key column | Index the referencing column |
| `OFFSET` pagination on large tables | Keyset pagination |
| `NOT IN (subquery)` with possible `NULL`s | `NOT EXISTS` |
| Dropping and recreating schemas in production | Versioned migrations |
| One superuser role for the application | Least-privilege roles |
| Concurrent "read then write" without locking | `SELECT ... FOR UPDATE`, or a single atomic statement |

## 14. Learning order

1. SQL basics: `SELECT`, `WHERE`, `ORDER BY`, `INSERT`, `UPDATE`, `DELETE`.
2. Table design: types, keys, constraints, normalisation.
3. Joins and aggregation.
4. Transactions and isolation.
5. Indexes and `EXPLAIN`.
6. CTEs, window functions, `jsonb`.
7. Roles, privileges and security.
8. Backup, restore and monitoring.
9. Optional: partitioning, replication, full-text search, extensions.

## 15. Practice questions

1. State the difference between SQL and PostgreSQL.
2. Why is `WHERE col = NULL` never true, and what should you write?
3. Compare `INNER JOIN` and `LEFT JOIN` using the `customers` and `orders` tables.
4. What is the difference between `WHERE` and `HAVING`?
5. Explain how MVCC lets readers and writers avoid blocking each other. What cleans up old row versions?
6. Which anomalies does Repeatable Read prevent in PostgreSQL that the SQL standard permits?
7. Two workers must claim different jobs from one queue table. Write the query.
8. Why can a foreign key column still need its own index?
9. When does the planner prefer a sequential scan over an index scan?
10. Why is `timestamptz` preferred over `timestamp`?
11. What is the difference between `pg_dump` and a base backup with WAL archiving?
12. Write a policy that lets each user see only their own rows.

## 16. References

- PostgreSQL Global Development Group. *PostgreSQL 16 Documentation*. <https://www.postgresql.org/docs/16/>
- ISO/IEC 9075. *Information technology — Database languages — SQL*.
- Codd, E. F. (1970). "A Relational Model of Data for Large Shared Data Banks." *Communications of the ACM* 13(6).
- Berenson, H., et al. (1995). "A Critique of ANSI SQL Isolation Levels." *ACM SIGMOD*.
- Ports, D. R. K., and Grittner, K. (2012). "Serializable Snapshot Isolation in PostgreSQL." *PVLDB* 5(12).
- Winand, M. *SQL Performance Explained*. <https://use-the-index-luke.com/>
