# SqlSkill — PostgreSQL + Supabase Best Practices

You are helping a developer build a database-intensive application on PostgreSQL, specifically with Supabase. Apply every rule in this file on every database-related task: schema design, query writing, migration authoring, and application-layer integration.

---

## Core thinking style

- **Think in sets, not loops.** Operate on entire sets of rows in a single SQL statement. Never suggest fetching rows one at a time in a loop when a set-based operation can handle it inside the database.
- **Ask for context before generating.** Before writing any non-trivial query or schema change, ask for: table names and column types, existing indexes, approximate row counts, and whether the workload is read-heavy or write-heavy. A query without this context is a guess.
- **The database is smarter than the application.** Constraints, computed values, and integrity checks belong in the database schema, not solely in application code. The database enforces them even when the application has a bug.

---

## Indexes

### When to add an index

- Index columns that appear in `WHERE`, `JOIN ON`, and `ORDER BY` clauses of real, frequent queries.
- Index foreign key columns. PostgreSQL does NOT create indexes on foreign keys automatically. Every un-indexed foreign key turns a simple parent lookup into a sequential scan of the child table.
- For columns used in equality-only lookups with high cardinality (user IDs, emails, UUIDs), the default `BTREE` index is correct.
- For `JSONB` columns where you query by key, use `GIN`. For full-text search vectors, use `GIN`. For large append-only tables ordered by time (event logs, audit trails), use `BRIN`.
- For `LIKE 'prefix%'` searches, a standard `BTREE` index works. For `LIKE '%substring%'`, you need a `GIN` index with the `pg_trgm` extension.

### When NOT to add an index

- Do not index every column by default. Every index is updated on every `INSERT`, `UPDATE`, and `DELETE` on that table. Over-indexing hurts write-heavy workloads significantly.
- Do not add an index because it "might help." Add it because a specific, identified query needs it.
- Do not assume the planner will use your index. On small tables or when a condition matches more than roughly 20% of rows, a sequential scan is cheaper. Check with `EXPLAIN ANALYZE`.

### Composite indexes

- Column order in a composite index matters. Put the highest-selectivity column first, or the column used for equality before the column used for range.
- `CREATE INDEX ON orders (user_id, created_at)` serves `WHERE user_id = $1 ORDER BY created_at DESC` efficiently. Reversing the column order does not.
- A covering index (`INCLUDE`) avoids a heap fetch entirely for index-only scans. Use it when the query selects only a few columns that can all fit in the index.

### Always verify with EXPLAIN ANALYZE

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE user_id = $1 ORDER BY created_at DESC LIMIT 20;
```

Look for: `Seq Scan` on large tables (missing index), large gap between `rows=estimate` and `actual rows` (stale statistics, run `ANALYZE`), `Buffers: shared hit` vs `shared read` (cache hit ratio).

---

## Transactions

### The non-negotiable rule

Wrap every operation that involves more than one write in an explicit transaction. If two writes must both succeed or both fail, they go inside `BEGIN ... COMMIT`. There is no other way to guarantee this.

```sql
BEGIN;
  UPDATE accounts SET balance = balance - 1000 WHERE id = $1;
  UPDATE accounts SET balance = balance + 1000 WHERE id = $2;
COMMIT;
```

If the application crashes between the two updates without a transaction, the money is gone. With a transaction, PostgreSQL rolls back automatically on crash.

### Keep transactions short

A long-running open transaction blocks VACUUM from cleaning dead row versions behind it. On high-churn tables, this causes table bloat and slows every subsequent query. Never open a transaction, then wait for user input or an external API call inside it. Do all external work before opening the transaction, then write fast.

```
BAD:  BEGIN → call Stripe API (200ms) → INSERT payment → COMMIT
GOOD: call Stripe API (200ms) → BEGIN → INSERT payment → COMMIT
```

### Isolation levels

- `READ COMMITTED` (the Postgres default) is correct for most OLTP workloads. Each statement sees a fresh snapshot of committed data.
- Use `REPEATABLE READ` when a transaction must see a consistent snapshot across multiple statements — for example, generating a report that joins several tables and must not see mid-report commits from other transactions.
- Use `SERIALIZABLE` only when correctness depends on no concurrent interference. Handle serialization failures (`ERROR 40001`) in application code with a retry loop.

### Savepoints for partial rollback

Use savepoints when a transaction has multiple stages and a later stage failing should not cancel earlier stages:

```sql
BEGIN;
  INSERT INTO orders (...) VALUES (...);
  SAVEPOINT after_order;
  INSERT INTO payments (...) VALUES (...);
  -- if payment insert fails:
  ROLLBACK TO after_order;
  -- order insert is still intact, transaction still open
COMMIT;
```

### Supabase-specific: transactions in client code

The Supabase JS client does not expose `BEGIN/COMMIT` directly. For multi-step transactional writes, use a Postgres function (RPC call) or a database migration that wraps the logic in a function. Do not attempt to serialize transactions across multiple `.from()` calls in application code — they are independent round-trips with no atomicity guarantee.

```sql
-- Define once in a migration
CREATE OR REPLACE FUNCTION transfer_funds(
  from_id uuid, to_id uuid, amount numeric
) RETURNS void LANGUAGE plpgsql AS $$
BEGIN
  UPDATE accounts SET balance = balance - amount WHERE id = from_id;
  UPDATE accounts SET balance = balance + amount WHERE id = to_id;
END;
$$;
```

```js
// Call from application
await supabase.rpc('transfer_funds', { from_id, to_id, amount })
```

---

## Connections and Connection Pools

### What a connection costs

Each PostgreSQL connection is a full OS process created via `fork()`. It consumes approximately 5 to 10 MB of RAM just to exist, before any query runs. Opening a new connection also requires a TCP handshake, TLS negotiation, and authentication round-trip — typically 50 to 200ms of latency on every cold open.

Allowing unconstrained connections will exhaust database memory before query load becomes the bottleneck. Supabase's managed instances have hard connection limits; hitting them causes new connections to fail with `FATAL: sorry, too many clients already`.

### Always use a connection pool

Never open a new database connection per HTTP request, per serverless invocation, or per background job. Use a pool.

**In serverless (Vercel, Netlify, Cloudflare Workers, AWS Lambda):**
- Connect through Supabase's Transaction Pooler on port `6543`, not the direct Postgres port `5432`.
- The Transaction Pooler (PgBouncer in transaction mode) multiplexes many short-lived application connections onto a small fixed number of real Postgres connections.
- Connection string pattern: use `?pgbouncer=true` when using Prisma; raw `pg` and `postgres.js` need no flag but must target port 6543.

```
Direct Postgres:     postgresql://user:pass@db.project.supabase.co:5432/postgres
Transaction Pooler:  postgresql://user:pass@db.project.supabase.co:6543/postgres
```

**In long-running servers (Node.js, Python, Rails):**
- Use the built-in pool of your ORM or driver: `pg.Pool` in node-postgres, `pool_size` in SQLAlchemy, `CONN_MAX_AGE` in Django.
- A pool size of 5 to 20 real connections is sufficient for most web application workloads. Start at 10 and tune based on `pg_stat_activity` monitoring.

**What NOT to do in serverless:**
```js
// WRONG — opens a new OS process on Postgres for every Lambda invocation
const client = new Client({ connectionString: process.env.DATABASE_URL })
await client.connect()
```

```js
// CORRECT — connect through the Transaction Pooler (port 6543)
import postgres from 'postgres'
const sql = postgres(process.env.DATABASE_URL) // pooler URL, not direct
```

### Session features break in transaction-mode pooling

PgBouncer in transaction mode returns the connection to the pool after each transaction, which means session-level state does not persist between transactions. The following features do not work reliably behind a transaction-mode pooler:

- Named prepared statements (`PREPARE ... AS`)
- `SET` session variables that need to persist
- Advisory locks held across transactions
- `LISTEN/NOTIFY` subscriptions

If you need these, connect through the direct port (5432) or use session-mode pooling, and account for the higher per-connection memory cost.

---

## Schema and constraints

- Enforce integrity at the database level: `NOT NULL`, `CHECK`, `UNIQUE`, and `FOREIGN KEY` constraints. Application-level validation is a supplement, not a replacement.
- `NULL` means unknown, not zero or empty string. `NULL = NULL` evaluates to `NULL`, not `true`. Always use `IS NULL` and `IS NOT NULL` in WHERE clauses. Be deliberate about which columns are nullable.
- Normalize by default. Denormalize only when a specific, measured read-performance problem justifies the consistency tradeoff.
- Use `uuid` primary keys (or `gen_random_uuid()`) for distributed systems. Use `bigserial` for simple sequential IDs where human readability matters.
- Always add `created_at TIMESTAMPTZ DEFAULT now()` and `updated_at TIMESTAMPTZ DEFAULT now()` to every entity table. Add a trigger to keep `updated_at` current automatically.

---

## Supabase-specific: Row Level Security

- Enable RLS on every table that holds user data. A table with RLS disabled is accessible to any authenticated user who can construct a direct Postgres connection or API call.
- Write RLS policies that reference `auth.uid()` to scope reads and writes to the authenticated user's own rows.
- Test RLS policies explicitly. Query the table with `SET role = authenticated` and a specific `SET request.jwt.claims.sub = '<user_id>'` to simulate a user's access in the SQL editor.
- Service-role keys bypass RLS entirely. Never expose the service-role key to the client. Use it only in trusted server-side code (API routes, edge functions) where you have verified authorization yourself.

---

## Query generation rules

1. Select only the columns needed. Never `SELECT *` in production code.
2. Always alias tables in multi-table queries.
3. Use parameterized queries or prepared statements. Never interpolate user input directly into a SQL string.
4. For bulk inserts, use a single multi-row `INSERT ... VALUES (...), (...)` rather than many individual inserts, or use `COPY` for large data loads.
5. Add a `RETURNING` clause on `INSERT`, `UPDATE`, and `DELETE` to get confirmation of what changed without a second round-trip.
6. Paginate with keyset pagination, not `OFFSET`. `OFFSET n` scans and discards `n` rows on every page request, which degrades linearly as `n` grows.

```sql
-- Keyset pagination (correct)
SELECT id, title, created_at FROM posts
WHERE created_at < $last_seen_created_at
ORDER BY created_at DESC
LIMIT 20;

-- OFFSET pagination (avoid on large tables)
SELECT id, title, created_at FROM posts
ORDER BY created_at DESC
LIMIT 20 OFFSET 10000; -- reads and discards 10,000 rows every time
```

7. For Supabase client queries, apply `.limit()` explicitly. The client does not add a default limit; an unbounded query on a large table will return the entire table.

---

## Migrations

- Every schema change goes through a migration file. Never alter a production table manually through the dashboard.
- Migrations must be reversible. Write both an `up` and a `down` step.
- Adding a column with a default value in PostgreSQL 11+ is safe and instant. In older versions it rewrites the table.
- Adding a `NOT NULL` constraint to an existing column with data requires a two-step migration: add the column as nullable and backfill, then add the constraint. Doing it in one step locks the table for the duration of the backfill.
- Adding an index concurrently avoids locking: `CREATE INDEX CONCURRENTLY idx_name ON table (column)`. Plain `CREATE INDEX` takes an exclusive lock for the duration.

---

## Monitoring checklist

- `pg_stat_activity` — active connections, their states, and long-running queries
- `pg_stat_user_tables` — `seq_scan` vs `idx_scan` ratio; high `seq_scan` on a large table means a missing index
- `pg_stat_user_indexes` — `idx_scan = 0` on an index means it is never used and can be dropped
- `pg_locks` — blocked queries; look for lock waits on the same table

---

## Source

Based on the official PostgreSQL documentation: https://www.postgresql.org/docs/current/
Supabase documentation: https://supabase.com/docs
