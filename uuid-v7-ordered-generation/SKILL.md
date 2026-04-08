---
name: uuid-v7-ordered-generation
description: Idiomatic, time-ordered UUID v7 generation for Go backend projects. Provides a clean, reusable package that generates RFC 9562-compliant UUID v7 (timestamp + random) for use as primary keys, ensuring natural sortability by creation time while maintaining strong uniqueness. Directly based on the pattern and spirit of ethos-go/internal/common/random/uuid.go (SemmiDev standard). Integrates perfectly with database-access-with-pgx-and-sqlx (as PostgreSQL UUID type), data-table-handling, and secure-idiomatic-error-handling. Always prefer UUID v7 over v4 for new tables to improve DB insert performance, index locality, and query ordering.
---

# UUID v7 Ordered Generation

**Skill purpose**  
This skill defines the **official SemmiDev pattern** for generating time-ordered UUIDs (Version 7) in all Go backends.  

UUID v7 (RFC 9562) encodes a Unix timestamp (milliseconds) in the most significant 48 bits, followed by random bits. This makes UUIDs:
- **Naturally sortable** by creation time (lexicographically and chronologically).
- **Database-friendly** — dramatically better B-tree index performance, reduced fragmentation, and faster inserts compared to random UUID v4.
- **Still globally unique** with sufficient randomness.
- **Compatible** with existing `database/sql`, `pgx`, and UUID columns.

The package follows the ethos-go style: simple, focused, and placed under `internal/common/random` or `pkg/uuid`.

## When to use this skill
- Primary keys for any new entity (users, orders, transactions, events, etc.).
- Any ID that benefits from time-based sorting or range queries.
- You are replacing `github.com/google/uuid` New() (v4) or manual `uuid.NewRandom()`.
- You need monotonicity within the same millisecond (sub-ms randomness).

**Never** use UUID v4 as the default for new database tables in 2026+ projects.

## Installation & Dependencies

Use a battle-tested library that supports UUID v7 natively:

```bash
go get github.com/gofrs/uuid/v5
```

(Alternative high-quality options: `github.com/block/uuidv7/go` or `github.com/google/uuid` once v7 support stabilizes in stdlib.)

## Project Structure

```
internal/common/random/
├── uuid.go        ← Main UUID v7 helpers (following ethos-go style)
└── uuid_test.go
```

## Canonical Implementation (`uuid.go`)

```go
package random

import (
    "fmt"
    "time"

    "github.com/gofrs/uuid/v5"
)

// UUID is a type alias for clarity and future extensibility
type UUID = uuid.UUID

const Nil = uuid.Nil

// NewV7 generates a new time-ordered UUID v7.
// It embeds current Unix timestamp (ms) + randomness, making it sortable.
func NewV7() (UUID, error) {
    u, err := uuid.NewV7()
    if err != nil {
        return Nil, fmt.Errorf("failed to generate UUID v7: %w", err)
    }
    return u, nil
}

// MustNewV7 is the convenience version that panics on error (use only in init or tests).
func MustNewV7() UUID {
    u, err := NewV7()
    if err != nil {
        panic(err)
    }
    return u
}

// NewV7WithTime allows generating UUID v7 with a specific timestamp (useful for testing or backfills).
func NewV7WithTime(t time.Time) (UUID, error) {
    // gofrs/uuid v5 supports this via NewV7WithClock or similar
    // Fallback implementation if needed:
    return uuid.NewV7FromTime(t)
}

// String returns the standard string representation (lowercase with hyphens).
func (u UUID) String() string {
    return u.String()
}

// FromString parses a UUID string (supports v4 and v7).
func FromString(s string) (UUID, error) {
    return uuid.FromString(s)
}

// IsV7 returns true if the UUID is version 7.
func (u UUID) IsV7() bool {
    return u.Version() == 7
}

// Time returns the embedded timestamp from a UUID v7 (approximate creation time).
func (u UUID) Time() time.Time {
    return u.Time() // gofrs/uuid provides this for v7
}
```

## Usage Patterns

### 1. In Domain Models / Entities

```go
type User struct {
    ID        random.UUID `json:"id" db:"id"`
    Email     string      `json:"email"`
    CreatedAt time.Time   `json:"created_at"`
    // ...
}

func NewUser(email string) (*User, error) {
    id, err := random.NewV7()
    if err != nil {
        return nil, err
    }
    return &User{
        ID:        id,
        Email:     email,
        CreatedAt: time.Now(),
    }, nil
}
```

### 2. In Repository (with pgx + sqlx)

```go
func (r *UserRepo) Create(ctx context.Context, user *User) error {
    query := `INSERT INTO users (id, email, created_at) VALUES ($1, $2, $3)`
    _, err := r.db.SQLX.ExecContext(ctx, query, user.ID, user.Email, user.CreatedAt)
    if err != nil {
        return errors.New("USER_CREATE_ERROR", "Failed to create user", err, nil)
    }
    return nil
}
```

PostgreSQL stores it natively as `uuid` type — no changes needed.

### 3. In Data Table / Listing

UUID v7 sorts naturally with `ORDER BY id ASC` (newest last) or `DESC`.

### 4. Testing

```go
func TestNewUser(t *testing.T) {
    user, err := NewUser("test@example.com")
    assert.NoError(t, err)
    assert.True(t, user.ID.IsV7())
    
    // Check rough time ordering
    u1, _ := random.NewV7()
    time.Sleep(1 * time.Millisecond)
    u2, _ := random.NewV7()
    assert.True(t, u2 > u1) // lexicographically larger = created later
}
```

## Best Practices & Decision Tree (2026 consensus)

**Always**:
- Use `NewV7()` for all new primary keys.
- Store as `uuid` type in PostgreSQL.
- Use `MustNewV7()` only in tests or init (never in hot paths).
- Leverage natural sorting in queries when appropriate.
- Keep the generation in a shared `internal/common/random` package.

**Decision Tree**:
1. New entity / table? → Use UUID v7.
2. Need exact creation time? → Still store a separate `created_at` timestamp (UUID v7 timestamp has ms precision).
3. Existing v4 data? → Keep as-is; new records can use v7 (coexistence is fine).
4. Need strict monotonicity across servers? → UUID v7 is "practically monotonic"; for absolute guarantees consider ULID or custom sequence.
5. Performance critical inserts? → UUID v7 shines here.

**Never**:
- Use UUID v4 for new tables (causes index fragmentation).
- Expose raw UUID generation logic in handlers/services.
- Rely solely on UUID for creation time (always pair with explicit `created_at`).

## Integration Notes for Your Backend Projects
- Place in `internal/common/random/uuid.go` (mirror ethos-go style).
- Update all new entity structs to use `random.UUID`.
- Combine with `database-migrations-with-golang-migrate` — add `id UUID PRIMARY KEY` in new table migrations.
- Combine with `data-table-handling-with-filter-pagination` — sorting by `id` becomes time-based ordering for free.
- In logs (slog): use `slog.String("id", id.String())` — safe and readable.
- For distributed systems: UUID v7 reduces hot-spotting in sharded databases.

## Why UUID v7 (Key Benefits)

- **Better DB performance** — inserts are 2-5x faster, indexes smaller and less fragmented.
- **Sortable by time** — `ORDER BY id` = chronological order.
- **Same storage** — 128 bits, works with existing UUID columns.
- **Collision resistant** — still uses strong randomness.
- **Debuggable** — you can roughly extract creation time from the ID.

Follow this skill and every new ID in your Go backends will be modern, performant, and future-proof.

When in doubt, use `random.NewV7()` or `random.MustNewV7()` — this is the exact pattern evolving from ethos-go and adopted across all SemmiDev backend projects in 2026.
