```markdown
---
name: database-access-with-pgx-and-sqlx
description: Idiomatic, high-performance, and fully context-aware PostgreSQL layer for Go backend projects using pgx/v5 (native driver + pgxpool) + sqlx (for struct scanning, named queries, and convenience). Includes clean Repository pattern, transaction boundaries managed via context propagation (transactor-style), automatic rollback on error, and perfect integration with secure-idiomatic-error-handling, structured-logging-with-slog, and http-error-handling-with-problem skills. Always use this abstraction instead of raw *sql.DB, direct pgx.Conn, or manual transaction handling.
---

# Database Access with pgx + sqlx (Context-Aware Transactions)

**Skill purpose**  
This skill defines the **official SemmiDev pattern** for PostgreSQL in all Go backends (2026 best practices).  

It combines:
- **pgx/v5** → native high-performance driver with full context support, Postgres-specific features, and `pgxpool` for connection pooling.
- **sqlx** → ergonomic struct scanning (`StructScan`), named queries, and `Get`/`Select` helpers on top of `database/sql` (via `pgx/stdlib` adapter).
- **Context propagation for transactions** → clean, signature-friendly way to run multi-repo operations inside a single transaction (inspired by the 2025 `transactor` pattern).

Result: Fully testable repositories, automatic transaction boundaries, zero context loss, graceful cancellation/timeouts, and seamless error wrapping into RFC 7807 Problems.

## When to use this skill
- Any database operation (CRUD, complex queries, reports).
- You need struct-to-row mapping without boilerplate.
- You need transactions spanning multiple repositories (e.g. create user + send welcome email atomically).
- You want context cancellation, request-scoped timeouts, and distributed tracing.
- You are replacing raw `database/sql`, plain `pgxpool`, or GORM/sqlc (unless sqlc is explicitly chosen).

**Never** use raw `*sql.DB`, `pgx.Conn` directly in services/handlers, or start transactions without context.

## Installation

```bash
go get github.com/jackc/pgx/v5
go get github.com/jackc/pgx/v5/pgxpool
go get github.com/jmoiron/sqlx
go get github.com/jackc/pgx/v5/stdlib  # for sqlx compatibility
```

## Project Structure (recommended)

```
internal/common/db/
├── db.go              ← Config, NewDB, DB interface
├── repository.go      ← base Repository interface + helpers
├── tx.go              ← context-based transaction propagation
├── queries/           ← sqlx queries or sql files (optional)
└── models/            ← internal DB structs (if needed)
```

## Core Concepts

### 1. DB Setup (`db.go`)

```go
package db

import (
    "context"
    "fmt"
    "time"

    "github.com/jackc/pgx/v5/pgxpool"
    "github.com/jmoiron/sqlx"
    "github.com/jackc/pgx/v5/stdlib"
)

type Config struct {
    DSN             string
    MaxConns        int           // default 10
    MinConns        int           // default 2
    MaxConnLifetime time.Duration // default 1h
    HealthCheckPeriod time.Duration
}

type DB struct {
    Pool *pgxpool.Pool
    SQLX *sqlx.DB          // sqlx wrapper for convenience
}

func New(cfg Config) (*DB, error) {
    poolConfig, err := pgxpool.ParseConfig(cfg.DSN)
    if err != nil {
        return nil, fmt.Errorf("parse DSN: %w", err)
    }

    poolConfig.MaxConns = int32(cfg.MaxConns)
    poolConfig.MinConns = int32(cfg.MinConns)
    poolConfig.MaxConnLifetime = cfg.MaxConnLifetime
    poolConfig.HealthCheckPeriod = cfg.HealthCheckPeriod

    pool, err := pgxpool.NewWithConfig(context.Background(), poolConfig)
    if err != nil {
        return nil, fmt.Errorf("create pgxpool: %w", err)
    }

    // sqlx adapter (uses pgx under the hood)
    stdDB := stdlib.OpenDBFromPool(pool)
    sqlxDB := sqlx.NewDb(stdDB, "pgx")

    return &DB{
        Pool: pool,
        SQLX: sqlxDB,
    }, nil
}

// Close for graceful shutdown
func (db *DB) Close() {
    db.Pool.Close()
}
```

### 2. Context-Based Transaction Propagation (`tx.go`)

```go
package db

import (
    "context"

    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/pgxpool"
)

type txKeyType int

const txKey txKeyType = iota

// WithTx injects a transaction into context (used by services)
func WithTx(ctx context.Context, tx pgx.Tx) context.Context {
    return context.WithValue(ctx, txKey, tx)
}

// GetQuerier returns sqlx-compatible querier: tx if present, otherwise pool
func (db *DB) GetQuerier(ctx context.Context) interface {
    sqlx.Ext
    sqlx.PreparerContext
    sqlx.QueryerContext
} {
    if tx, ok := ctx.Value(txKey).(pgx.Tx); ok {
        // Wrap pgx.Tx with sqlx for scanning
        return sqlx.NewDb(stdlib.OpenDBFromConn(tx.Conn()), "pgx").Ext()
    }
    return db.SQLX
}

// RunInTx executes fn inside a transaction (service-level boundary)
func (db *DB) RunInTx(ctx context.Context, fn func(ctx context.Context) error) error {
    tx, err := db.Pool.Begin(ctx)
    if err != nil {
        return fmt.Errorf("begin tx: %w", err)
    }
    defer func() {
        if p := recover(); p != nil {
            tx.Rollback(ctx)
            panic(p)
        }
    }()

    ctx = WithTx(ctx, tx)

    if err := fn(ctx); err != nil {
        if rbErr := tx.Rollback(ctx); rbErr != nil {
            return fmt.Errorf("rollback: %v (original: %w)", rbErr, err)
        }
        return err
    }

    return tx.Commit(ctx)
}
```

### 3. Repository Pattern (`repository.go`)

```go
package db

import "context"

type Repository[T any] interface {
    // All methods take context + return domain types + SafeError compatible errors
    GetByID(ctx context.Context, id string) (*T, error)
    List(ctx context.Context, filter Filter) ([]T, error)
    Create(ctx context.Context, entity *T) error
    Update(ctx context.Context, entity *T) error
    // ...
}

// Example UserRepo (implements Repository[User])
type UserRepo struct {
    db *DB
}

func NewUserRepo(db *DB) *UserRepo { return &UserRepo{db: db} }

func (r *UserRepo) GetByID(ctx context.Context, id string) (*User, error) {
    q := r.db.GetQuerier(ctx) // tx or pool automatically

    var u User
    err := q.GetContext(ctx, &u, `SELECT * FROM users WHERE id = $1`, id)
    if err != nil {
        // wrap with secure error skill
        return nil, errors.New("USER_NOT_FOUND", "User not found", err, map[string]string{"user_id": id})
    }
    return &u, nil
}
```

## Recommended Usage Patterns

### 1. Setup (in main)
```go
var Database *db.DB

func initDB() {
    cfg := db.Config{ DSN: os.Getenv("DATABASE_URL"), /* ... */ }
    var err error
    Database, err = db.New(cfg)
    if err != nil { panic(err) }
}
```

### 2. Single operation (no tx)
```go
func GetUserHandler(w http.ResponseWriter, r *http.Request) {
    user, err := userRepo.GetByID(r.Context(), id)
    if err != nil {
        p := errors.ToProblem(err, r.URL.Path)
        p.Write(w)
        return
    }
    // success
}
```

### 3. Multi-repo transaction (service layer)
```go
func CreateUserWithWelcome(ctx context.Context, req CreateUserRequest) error {
    return Database.RunInTx(ctx, func(ctx context.Context) error {
        // All repos automatically use the same tx via context
        if err := userRepo.Create(ctx, &user); err != nil {
            return err
        }
        if err := tokenRepo.Create(ctx, &token); err != nil {
            return err
        }

        // Email can be sent outside tx (or use outbox pattern)
        return emailService.SendWelcome(ctx, user.Email)
    })
}
```

### 4. Logging integration (in repo or middleware)
```go
slog.InfoContext(ctx, "db query executed",
    slog.String("query", "SELECT ..."),
    slog.String("user_id", id),
)
```

## Best Practices & Decision Tree (2026 consensus)

**Always**:
- Pass `context.Context` as first argument to **every** DB method.
- Use `db.GetQuerier(ctx)` so code works inside/outside transactions.
- Start transactions only at the service layer (`RunInTx`).
- Use `defer tx.Rollback` + check commit error.
- Return domain models (not DB structs) from repositories.
- Wrap all DB errors with `secure-idiomatic-error-handling`.

**Decision Tree**:
1. Single query? → Call repo method directly (uses pool).
2. Multiple operations that must be atomic? → Wrap in `Database.RunInTx`.
3. Need Postgres-specific feature (COPY, LISTEN/NOTIFY)? → Use `db.Pool` directly in that repo method.
4. Testing? → Mock the Repository interface (or inject fake DB).
5. High throughput? → Tune pool config + use sqlx named queries.

**Never**:
- Start/commit/rollback manually in repositories.
- Ignore context cancellation.
- Use positional args without named queries when possible.
- Leak connection (always let pool manage Acquire/Release).

## Integration Notes for Your Backend Projects
- Place in `internal/common/db` and inject `*db.DB` into all repos.
- Combine with `secure-idiomatic-error-handling` → all DB errors become safe Problems.
- Combine with `structured-logging-with-slog` → add `request_id` automatically.
- For local/dev: use Docker Postgres + connection string.
- Graceful shutdown: call `Database.Close()` on SIGTERM.
- Future-proof: easy to swap to sqlc later (same repository interfaces).

Follow this skill and every database interaction in your Go backends will be:
- High-performance (native pgx)
- Convenient (sqlx scanning)
- Transaction-safe (context propagation)
- Testable & observable
- Production-ready (pool, context, errors)

When in doubt, copy the `RunInTx` + `GetQuerier` pattern above — this is the exact design now used across all SemmiDev backend projects after adopting 2025/2026 pgx + sqlx best practices.
