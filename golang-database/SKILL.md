---
name: golang-database
description: "Comprehensive guide for Go database access with pgx and PostgreSQL. Covers parameterized queries, struct scanning, NULLable columns, error patterns, transactions, isolation levels, SELECT FOR UPDATE, connection pool, batch processing, context propagation, and goose migrations. Use when writing, reviewing, or debugging Go code that interacts with PostgreSQL."
license: MIT
metadata:
  original-author: samber
  original-repo: https://github.com/samber/cc-skills-golang/tree/main/skills/golang-database
  version: "1.1.2"
  adapted-by: openserbia
  adapted-changes: "Focused on pgx + PostgreSQL, replaced golang-migrate with goose, added RW/RO pool pattern, aligned with openserbia conventions"
---

**Persona:** You are a Go backend engineer who writes safe, explicit, and observable database code. You treat SQL as a first-class language — no ORMs, no magic — and you catch data integrity issues at the boundary, not deep in the application.

**Modes:**

- **Write mode** — generating new repository functions, query helpers, or transaction wrappers: follow the skill's sequential instructions; launch a background agent to grep for existing query patterns and naming conventions in the codebase before generating new code.
- **Review/debug mode** — auditing or debugging existing database code: use a sub-agent to scan for missing `rows.Close()`, un-parameterized queries, missing context propagation, and absent error checks in parallel with reading the business logic.

# Go Database Best Practices

Use `pgx` for PostgreSQL — never an ORM. Raw SQL with parameterized queries gives you full control, predictable performance, and debuggable code.

## Best Practices Summary

1. **Use pgx, not ORMs** — ORMs hide SQL, generate unpredictable queries, and make debugging harder
2. Queries MUST use parameterized placeholders (`$1, $2`) — NEVER concatenate user input into SQL strings
3. Context MUST be passed to all database operations
4. `pgx.ErrNoRows` MUST be handled explicitly — distinguish "not found" from real errors using `errors.Is`
5. Rows MUST be closed after iteration — `defer rows.Close()` immediately after `Query` calls
6. **Use transactions for multi-statement operations** — wrap related writes in `Begin`/`Commit`
7. **Use `SELECT ... FOR UPDATE`** when reading data you intend to modify — prevents race conditions
8. **Handle NULLable columns** with pointer fields (`*string`, `*int`) or `sql.NullXxx` types
9. Connection pool MUST be configured — `MaxConns`, `MaxConnLifetime`, `MaxConnIdleTime`
10. **Use goose for migrations** — versioned SQL files, never hand-rolled or AI-generated migration runners
11. **Batch operations in reasonable sizes** — not row-by-row, not millions at once
12. **Use RW/RO pool split** — reads go to read-only pool, writes go to read-write pool
13. **Avoid hidden SQL features** — do not rely on triggers (except `updated_at`), views, stored procedures, or row-level security in application code

## Library Choice

| Library | Use when |
| --- | --- |
| `pgx` (preferred) | All PostgreSQL projects — 30-50% faster, native types, COPY, LISTEN, arrays |
| `database/sql` + pgx stdlib | Need `database/sql` interface compatibility |
| GORM/ent | **Never** |

**Why NOT ORMs:**

- Unpredictable query generation — N+1 problems you cannot see in code
- Magic hooks and callbacks make debugging harder
- Schema migrations coupled to application code
- Learning the ORM API is harder than learning SQL, and the abstraction leaks

## Connection Pool — RW/RO Split

Use separate connection pools for reads and writes:

```go
type ConnProvider interface {
    RW() *pgxpool.Pool  // Write operations
    RO() *pgxpool.Pool  // Read operations
    Close()
}
```

- Use `provider.RO()` for all SELECT queries
- Use `provider.RW()` for INSERT/UPDATE/DELETE
- If `DATABASE_URL_RO` is not set, RO pool reuses the RW pool (automatic fallback)
- Configure `MaxConns` and `MaxConnLifetime` on both pools

```go
cfg, _ := pgxpool.ParseConfig(url)
cfg.MaxConns = 4
cfg.MaxConnLifetime = 30 * time.Minute
pool, _ := pgxpool.NewWithConfig(ctx, cfg)
```

## Parameterized Queries

```go
// VERY BAD — SQL injection vulnerability
query := fmt.Sprintf("SELECT * FROM users WHERE email = '%s'", email)

// Good — parameterized (pgx uses $1, $2, ...)
var user User
err := pool.QueryRow(ctx,
    "SELECT id, name, email FROM users WHERE email = $1", email,
).Scan(&user.ID, &user.Name, &user.Email)
```

### Dynamic column names

Never interpolate column names from user input. Use an allowlist:

```go
allowed := map[string]bool{"name": true, "email": true, "created_at": true}
if !allowed[sortCol] {
    return fmt.Errorf("invalid sort column: %s", sortCol)
}
query := fmt.Sprintf("SELECT id, name, email FROM users ORDER BY %s", sortCol)
```

## Struct Scanning and NULLable Columns

With pgx, scan fields manually or use `pgx.RowToStructByName`:

```go
// Manual scan
var user User
err := pool.QueryRow(ctx,
    "SELECT id, name, email, bio FROM users WHERE id = $1", id,
).Scan(&user.ID, &user.Name, &user.Email, &user.Bio)

// For NULLable columns, use pointers
type User struct {
    ID    int64
    Name  string
    Email string
    Bio   *string  // nullable
}
```

For NULL pointer helpers in repositories:

```go
func getStringPtr(v sql.NullString) *string {
    if v.Valid {
        return &v.String
    }
    return nil
}
```

See [Scanning Reference](./references/scanning.md) for more patterns.

## Error Handling

```go
func (r *UserRepo) GetByID(ctx context.Context, id int64) (domain.User, error) {
    var user domain.User
    err := r.db.RO().QueryRow(ctx,
        "SELECT id, name, email FROM users WHERE id = $1", id,
    ).Scan(&user.ID, &user.Name, &user.Email)

    if errors.Is(err, pgx.ErrNoRows) {
        return user, domain.ErrNotFound  // translate to domain error
    }
    return user, err
}
```

### Always close rows

```go
rows, err := pool.Query(ctx, "SELECT id, name FROM users")
if err != nil {
    return fmt.Errorf("querying users: %w", err)
}
defer rows.Close()

for rows.Next() {
    // ...
}
if err := rows.Err(); err != nil {
    return fmt.Errorf("iterating users: %w", err)
}
```

### Common database error patterns

| Error | How to detect | Action |
| --- | --- | --- |
| Row not found | `errors.Is(err, pgx.ErrNoRows)` | Return `domain.ErrNotFound` |
| Unique constraint | PostgreSQL error code `23505` | Return conflict error |
| Connection refused | `err != nil` on `pool.Ping` | Fail fast, log, retry with backoff |
| Serialization failure | PostgreSQL error code `40001` | Retry the entire transaction |
| Context canceled | `errors.Is(err, context.Canceled)` | Stop processing, propagate |

## Context Propagation

Always pass context to database operations:

```go
// Bad — no context
pool.Query("SELECT ...")

// Good — respects cancellation and timeouts
pool.Query(ctx, "SELECT ...")
```

## Transactions, Isolation Levels, and Locking

For transaction patterns, isolation levels, `SELECT FOR UPDATE`, and locking variants, see [Transactions](./references/transactions.md).

Basic pattern with pgx:

```go
tx, err := pool.Begin(ctx)
if err != nil {
    return fmt.Errorf("begin tx: %w", err)
}
defer tx.Rollback(ctx) // no-op if committed

_, err = tx.Exec(ctx, "UPDATE accounts SET balance = balance - $1 WHERE id = $2", amount, fromID)
if err != nil {
    return fmt.Errorf("debit: %w", err)
}

_, err = tx.Exec(ctx, "UPDATE accounts SET balance = balance + $1 WHERE id = $2", amount, toID)
if err != nil {
    return fmt.Errorf("credit: %w", err)
}

return tx.Commit(ctx)
```

## Migrations — Goose

Use [goose](https://github.com/pressly/goose) for database migrations. SQL files are versioned in `migrations/` directory.

### File conventions

- Filename: `YYYYMMDDHHMMSS_description.sql` (timestamp-based ordering)
- Always provide both `Up` and `Down` migrations
- Wrap in `-- +goose StatementBegin` / `-- +goose StatementEnd`
- Use `IF NOT EXISTS` / `IF EXISTS` for idempotency

```sql
-- +goose Up
-- +goose StatementBegin
CREATE TABLE IF NOT EXISTS users (
    id         BIGINT      PRIMARY KEY,
    name       TEXT        NOT NULL DEFAULT '',
    email      TEXT        NOT NULL DEFAULT '',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

DROP TRIGGER IF EXISTS trg_users_updated_at ON users;
CREATE TRIGGER trg_users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION set_updated_at();
-- +goose StatementEnd

-- +goose Down
-- +goose StatementBegin
DROP TRIGGER IF EXISTS trg_users_updated_at ON users;
DROP TABLE IF EXISTS users;
-- +goose StatementEnd
```

### Commands

```bash
task migrate:up              # Apply all pending migrations
task migrate:down            # Roll back last migration
task migrate:new name=foo    # Create new migration file
goose -dir migrations postgres "$DATABASE_URL" status  # Check migration status
```

### Rules

- Use `TIMESTAMPTZ` for all timestamp columns, default `now()`
- Create `updated_at` triggers on mutable tables (using shared `set_updated_at()` function)
- Down migration must reverse Up completely (drop in reverse order)
- Migration SQL should be written and reviewed by humans — AI does not understand production data volumes and access patterns
- Migrations run automatically on container start via `entrypoint.sh`

## Avoid Hidden SQL Features

Do not rely on triggers (except `updated_at`), views, materialized views, stored procedures, or row-level security in application code — they create invisible side effects and make debugging impossible. Keep SQL explicit and visible in Go where it can be tested and version-controlled.

## Schema Creation

**Be cautious with AI-generated schemas.** AI-generated schemas can be subtly wrong — missing indexes, incorrect column types, bad normalization. Schema design requires understanding data volumes, access patterns, and business constraints. Always review carefully.

## Deep Dives

- **[Transactions](./references/transactions.md)** — Transaction boundaries, isolation levels, deadlock prevention, `SELECT FOR UPDATE`
- **[Testing Database Code](./references/testing.md)** — Mock connections, integration tests with containers, fixtures, schema setup/teardown
- **[Database Performance](./references/performance.md)** — Connection pool sizing, batch processing, indexing strategy, query optimization
- **[Struct Scanning](./references/scanning.md)** — Struct tags, NULLable column handling, JSON marshaling patterns

## Attribution

Based on [samber/cc-skills-golang@golang-database](https://github.com/samber/cc-skills-golang/tree/main/skills/golang-database) (MIT License). Adapted for openserbia Go stack (pgx-focused, goose migrations, RW/RO pool split).
