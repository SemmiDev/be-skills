---
name: rich-domain-model-design-ddd
description: Idiomatic, rich Domain-Driven Design (DDD) tactical patterns for Go backend projects. Focuses on non-anemic models where Entities and Value Objects encapsulate both state and behavior, enforce invariants through private fields + constructor functions, and use Aggregate Roots to protect consistency. Inspired by Eric Evans' DDD, Vaughn Vernon's IDDD, and practical Go implementations (ThreeDotsLabs wild-workouts, goddd samples, and 2026 community consensus). Integrates seamlessly with UUID v7, config loading, database repositories, and all previous skills. Always design domain objects with behavior (methods expressing ubiquitous language) instead of anemic data structs.
---

# Rich Domain Model Design (DDD Tactical Patterns)

**Skill purpose**  
This skill defines the **official SemmiDev pattern** for building rich, non-anemic domain models in Go using DDD tactical building blocks.  

In contrast to the common *anemic domain model* (plain structs with getters/setters and logic scattered in services), a rich domain model:
- Encapsulates **state + behavior** inside Entities and Value Objects.
- Protects **invariants** (business rules) at construction and through domain methods.
- Uses **ubiquitous language** in method names (`PlaceOrder`, `ActivateAccount`, `ApplyDiscount`).
- Defines **Aggregates** with a single Aggregate Root that controls consistency.

This makes the domain code readable by business experts, testable in isolation, and resilient to change — exactly the goal of DDD.

## When to use this skill
- Complex business logic where rules and invariants matter (fintech, e-commerce, SaaS core domains).
- You want domain code that reflects real business language and rules.
- You are evolving from CRUD/anemic models to maintainable, evolvable systems.
- You need clear boundaries for testing, refactoring, and team collaboration.

**Never** use plain data structs (`type User struct { ... }`) with all logic in services/repositories — this is the anemic anti-pattern.

## Project Structure (idiomatic Go + DDD)

```
internal/
├── domain/                  ← Core rich domain (bounded context)
│   ├── order/               ← One aggregate per package (recommended)
│   │   ├── aggregate.go     ← Order (Aggregate Root)
│   │   ├── entity.go        ← OrderLine (child entity)
│   │   ├── valueobject.go   ← Money, OrderStatus, etc.
│   │   ├── factory.go       ← NewOrder, etc.
│   │   └── event.go         ← Domain events (optional)
│   └── user/                ← Another bounded context / aggregate
├── application/             ← Services that orchestrate multiple aggregates
├── infrastructure/          ← Repositories, DB adapters
└── ...                      ← Your existing packages
```

**Key rule**: One aggregate = one package. Keep the aggregate small and focused.

## Core Concepts & Idiomatic Go Patterns

### 1. Value Object (Immutable, Equality by Value)

```go
// internal/domain/order/valueobject/money.go
package valueobject

import "errors"

type Money struct {
    amount   int64  // unexported
    currency string // unexported
}

func NewMoney(amount int64, currency string) (Money, error) {
    if amount < 0 {
        return Money{}, errors.New("money amount cannot be negative")
    }
    if currency == "" {
        return Money{}, errors.New("currency is required")
    }
    return Money{amount: amount, currency: currency}, nil
}

// Business methods
func (m Money) Add(other Money) (Money, error) {
    if m.currency != other.currency {
        return Money{}, errors.New("currency mismatch")
    }
    return NewMoney(m.amount+other.amount, m.currency)
}

func (m Money) IsZero() bool { return m.amount == 0 }

// Value equality (Go idiom: value receiver + comparable fields)
func (m Money) Equals(other Money) bool {
    return m.amount == other.amount && m.currency == other.currency
}
```

### 2. Entity (Identity + Mutable Behavior)

```go
// internal/domain/order/entity/order_line.go
package entity

import (
    "yourproject/internal/domain/order/valueobject"
    "github.com/gofrs/uuid/v5"
)

type OrderLine struct {
    id       uuid.UUID          // identity
    productID uuid.UUID
    quantity int
    unitPrice valueobject.Money
}

func NewOrderLine(productID uuid.UUID, quantity int, unitPrice valueobject.Money) (*OrderLine, error) {
    if quantity <= 0 {
        return nil, errors.New("quantity must be positive")
    }
    return &OrderLine{
        id:        random.MustNewV7(), // from your UUID skill
        productID: productID,
        quantity:  quantity,
        unitPrice: unitPrice,
    }, nil
}

// Domain behavior
func (ol *OrderLine) IncreaseQuantity(delta int) error {
    if delta <= 0 {
        return errors.New("delta must be positive")
    }
    ol.quantity += delta
    return nil
}

func (ol OrderLine) LineTotal() valueobject.Money {
    // implementation using Money.Add...
}
```

### 3. Aggregate Root (Controls the Aggregate)

```go
// internal/domain/order/aggregate/order.go
package order

import (
    "time"
    "yourproject/internal/domain/order/entity"
    "yourproject/internal/domain/order/valueobject"
    "github.com/gofrs/uuid/v5"
)

type Order struct {
    id         uuid.UUID
    customerID uuid.UUID
    lines      []entity.OrderLine // private
    status     valueobject.OrderStatus
    createdAt  time.Time
    // no public setters
}

func NewOrder(customerID uuid.UUID, lines []entity.OrderLine) (*Order, error) {
    if len(lines) == 0 {
        return nil, errors.New("order must have at least one line")
    }
    o := &Order{
        id:         random.MustNewV7(),
        customerID: customerID,
        lines:      lines,
        status:     valueobject.OrderStatusDraft,
        createdAt:  time.Now(),
    }
    if err := o.validateInvariants(); err != nil {
        return nil, err
    }
    return o, nil
}

// Rich behavior (ubiquitous language)
func (o *Order) AddLine(line entity.OrderLine) error {
    o.lines = append(o.lines, line)
    return o.validateInvariants()
}

func (o *Order) Confirm() error {
    if o.status != valueobject.OrderStatusDraft {
        return errors.New("only draft orders can be confirmed")
    }
    o.status = valueobject.OrderStatusConfirmed
    // raise domain event if needed
    return nil
}

func (o Order) validateInvariants() error {
    // enforce aggregate consistency
    if len(o.lines) == 0 {
        return errors.New("order must have lines")
    }
    // total calculation, etc.
    return nil
}

// Expose read-only views (no getters for mutable state)
func (o Order) ID() uuid.UUID             { return o.id }
func (o Order) Lines() []entity.OrderLine { return o.lines } // return copy if needed
```

### 4. Factory (for Complex Creation)

```go
// internal/domain/order/factory/order_factory.go
func CreateOrderFromCart(cart Cart, customerID uuid.UUID) (*Order, error) {
    // map cart items → OrderLines with validation
    // ...
    return NewOrder(customerID, lines)
}
```

### 5. Repository Interface (infrastructure layer)

```go
// internal/application/order/repository.go
type OrderRepository interface {
    Save(ctx context.Context, order *order.Order) error
    FindByID(ctx context.Context, id uuid.UUID) (*order.Order, error)
    // List uses your filter skill
}
```

## Best Practices & Decision Tree (2026 Go + DDD consensus)

**Always**:
- Use **unexported fields** + constructor functions (`NewXXX`) to enforce invariants.
- Put **business logic inside** the aggregate root/entity/value object.
- Name methods with **ubiquitous language** (`ConfirmOrder`, `ApplyCoupon`, `CalculateShipping`).
- Keep aggregates small and consistent (one transaction = one aggregate).
- Use **domain events** for side effects (published after successful aggregate change).
- Test domain objects in isolation (no DB, no HTTP).

**Decision Tree**:
1. Has identity + lifecycle? → **Entity**.
2. Immutable + equality by value? → **Value Object**.
3. Cluster of related objects needing consistency? → **Aggregate** with **Root**.
4. Complex creation? → **Factory**.
5. Cross-aggregate logic? → **Domain Service** (thin, no state).

**Never**:
- Expose mutable fields or public setters.
- Put business rules in application services or repositories.
- Make everything an aggregate root (keep aggregates small).
- Use ORM auto-mapping that bypasses constructors (manual mapping in repo).

## Integration Notes for Your Backend Projects
- Place rich domain in `internal/domain/` (bounded contexts as sub-packages).
- Use your **UUID v7** as identity for all entities.
- Repositories live in `infrastructure/` and map to/from domain models.
- Application services orchestrate (call domain methods, then repo.Save).
- Combine with **config loading**, **slog**, **problem errors**, **Chi handlers**.
- For atstex-lab-style projects: this replaces any anemic structs with rich domain objects.

Follow this skill and your domain layer will be:
- Expressive and business-aligned
- Invariant-safe by design
- Testable and evolvable
- Truly idiomatic Go + DDD

When in doubt, start with **Value Objects** and **constructor functions** — this is the exact rich-domain approach now used across modern SemmiDev backends and the broader Go DDD community.
