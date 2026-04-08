---
name: money-decimal-handling-with-shopspring-decimal
description: Production-grade, precise monetary value handling using github.com/shopspring/decimal. Treats money as a rich Value Object (immutable, invariant-protected) following DDD best practices. Enforces frontend → string input (never float), JSON marshaling as string, PostgreSQL NUMERIC storage, and full arithmetic safety. Integrates with rich-domain-model-design-ddd, clean-idiomatic-config-loading, database-access-with-pgx-and-sqlx, and http-error-handling-with-problem. Never use float64 or int for money.
---

# Money & Decimal Handling with shopspring/decimal

**Skill purpose**  
This skill defines the **official SemmiDev pattern** for handling monetary values using `github.com/shopspring/decimal` — the most battle-tested, immutable, arbitrary-precision decimal library in the Go ecosystem (widely used in fintech, e-commerce, and high-stakes systems).

Key rules enforced by this skill:
- Money is a **rich Value Object** (not a primitive).
- Frontend **always sends amount as string** (e.g. `"1250.75"`, never `1250.75` float).
- All calculations are exact (no floating-point precision loss).
- JSON serialization/deserialization always uses string to protect frontend consumers.
- PostgreSQL stores as `NUMERIC` (exact precision) or `TEXT` (string).
- Full invariant protection (non-negative amount, currency consistency).

## When to use this skill
- Any price, amount, balance, fee, tax, or financial field.
- You need safe addition, multiplication, division, rounding, or comparison.
- You want to prevent the classic "0.1 + 0.2 = 0.30000000000000004" bugs.
- You are building or refactoring any domain that deals with money.

**Never** use `float64`, `int`, or raw `decimal.Decimal` directly in your domain or handlers.

## Installation

```bash
go get github.com/shopspring/decimal
```

## Project Structure

```
internal/domain/
└── valueobject/
    └── money.go          ← Core Money Value Object
```

## Canonical Implementation (`internal/domain/valueobject/money.go`)

```go
package valueobject

import (
    "database/sql/driver"
    "encoding/json"
    "fmt"

    "github.com/shopspring/decimal"
    "yourproject/internal/common/errors" // your secure error skill
)

var (
    ZeroMoney = Money{decimal: decimal.Zero}
)

// Money is an immutable Value Object representing a monetary amount.
// It wraps shopspring/decimal and enforces domain invariants.
type Money struct {
    decimal decimal.Decimal // unexported to protect invariants
    currency string          // optional but recommended
}

// NewMoney creates a new Money from a string (the only safe input method).
// Frontend must always send amount as string.
func NewMoney(amountStr, currency string) (Money, error) {
    if amountStr == "" {
        return ZeroMoney, fmt.Errorf("amount string is required")
    }

    d, err := decimal.RequireFromString(amountStr)
    if err != nil {
        return ZeroMoney, errors.New("INVALID_MONEY_FORMAT", "Invalid money format", err, nil)
    }

    if d.IsNegative() {
        return ZeroMoney, fmt.Errorf("money amount cannot be negative")
    }

    return Money{
        decimal:  d,
        currency: currency,
    }, nil
}

// MustNewMoney panics on error (safe to use in tests/factories).
func MustNewMoney(amountStr, currency string) Money {
    m, err := NewMoney(amountStr, currency)
    if err != nil {
        panic(err)
    }
    return m
}

// Business methods (rich behavior)
func (m Money) Add(other Money) (Money, error) {
    if m.currency != other.currency && m.currency != "" && other.currency != "" {
        return ZeroMoney, fmt.Errorf("currency mismatch")
    }
    return Money{decimal: m.decimal.Add(other.decimal), currency: m.currency}, nil
}

func (m Money) Subtract(other Money) (Money, error) {
    if m.currency != other.currency && m.currency != "" && other.currency != "" {
        return ZeroMoney, fmt.Errorf("currency mismatch")
    }
    result := m.decimal.Sub(other.decimal)
    if result.IsNegative() {
        return ZeroMoney, fmt.Errorf("result would be negative")
    }
    return Money{decimal: result, currency: m.currency}, nil
}

func (m Money) Multiply(factor decimal.Decimal) Money {
    return Money{decimal: m.decimal.Mul(factor), currency: m.currency}
}

func (m Money) IsZero() bool {
    return m.decimal.IsZero()
}

func (m Money) String() string {
    return m.decimal.String()
}

func (m Money) Currency() string {
    return m.currency
}

// JSON marshaling — always as string (critical for FE safety)
func (m Money) MarshalJSON() ([]byte, error) {
    return json.Marshal(m.decimal.String())
}

func (m *Money) UnmarshalJSON(data []byte) error {
    var s string
    if err := json.Unmarshal(data, &s); err != nil {
        return err
    }
    money, err := NewMoney(s, "")
    if err != nil {
        return err
    }
    *m = money
    return nil
}

// Database driver support (for pgx + sqlx)
func (m Money) Value() (driver.Value, error) {
    return m.decimal.String(), nil
}

func (m *Money) Scan(src any) error {
    switch v := src.(type) {
    case string:
        money, err := NewMoney(v, "")
        if err != nil {
            return err
        }
        *m = money
        return nil
    case []byte:
        return m.Scan(string(v))
    default:
        return fmt.Errorf("cannot scan %T into Money", src)
    }
}
```

## Recommended Usage Patterns

### 1. In Rich Domain Models
```go
type Order struct {
    id      uuid.UUID
    total   valueobject.Money
    // ...
}

func NewOrder(...) (*Order, error) {
    total, err := valueobject.NewMoney("1250.75", "IDR")
    if err != nil {
        return nil, err
    }
    // ...
}
```

### 2. In Handlers (Frontend always sends string)
```go
type CreateOrderRequest struct {
    Total string `json:"total" validate:"required"` // string, not float
}

func (h *Handler) CreateOrder(w http.ResponseWriter, r *http.Request) {
    var req CreateOrderRequest
    // ... decode + validate

    total, err := valueobject.NewMoney(req.Total, "IDR")
    if err != nil {
        problem.New(problem.BadRequest, problem.WithDetail(err.Error())).Write(w)
        return
    }
    // ...
}
```

### 3. In Repository (PostgreSQL)
Use `NUMERIC` column type:

```sql
-- migration
ALTER TABLE orders ADD COLUMN total NUMERIC(20,4) NOT NULL;
```

In repo:

```go
err := db.SQLX.GetContext(ctx, &order.Total, `SELECT total FROM orders WHERE id = $1`, id)
```

## Best Practices & Decision Tree

**Always**:
- Accept money from frontend **only as string**.
- Return money in JSON **only as string**.
- Use `NewMoney` / `MustNewMoney` for all creation.
- Store as `NUMERIC` in Postgres (or `TEXT` if you prefer string).
- Never perform arithmetic with `float64`.

**Decision Tree**:
1. Simple amount? → `valueobject.NewMoney("123.45", "IDR")`
2. Need currency? → Include currency field (or create `MoneyWithCurrency` type).
3. Need rounding? → Use `decimal.Decimal` methods (`Round`, `Truncate`, etc.).
4. Need tax / discount calculation? → Use the `Add`, `Subtract`, `Multiply` methods on `Money`.

**Never**:
- Use `float64` anywhere near money.
- Accept number (float) from JSON for money fields.
- Do manual string parsing outside `NewMoney`.

## Integration Notes
- Place `money.go` in `internal/domain/valueobject/`.
- Combine with **rich-domain-model-design-ddd** — `Money` is a perfect Value Object.
- Combine with **designing-custom-validation-helper** — add custom validator tag for money strings.
- Combine with **data-table-handling-with-filter-pagination** — amounts render as strings.
- Combine with **http-error-handling-with-problem** — invalid money format becomes clean Problem response.

Follow this skill and every monetary value in your Go backends will be:
- Mathematically precise
- Frontend-safe (string-only)
- Domain-rich and expressive
- Production-proven (same pattern used by major fintech systems)

When in doubt, always create money with `valueobject.NewMoney(amountStr, currency)` and let the Value Object protect the invariants.
