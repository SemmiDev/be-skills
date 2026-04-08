```markdown
---
name: data-table-handling-with-filter-pagination
description: Canonical, production-grade data table handling for Go backend APIs using the exact filter/pagination pattern from ethos-go/internal/common/model/filter.go (enhanced with 2026 best practices). Provides a reusable Filter struct (pagination, keyword search, sorting, date ranges, status filters), Paging metadata, safe dynamic query building with pgx + sqlx, column whitelisting, and perfect integration with database-access-with-pgx-and-sqlx, secure-idiomatic-error-handling, and structured-logging-with-slog. Always use this for any list/table endpoint instead of manual query params or ad-hoc pagination logic.
---

# Data Table Handling with Filter & Pagination

**Skill purpose**  
This skill establishes the **official SemmiDev pattern** for all list/table endpoints in Go backends.  

It is directly based on (and improves) the proven design from `ethos-go/internal/common/model/filter.go`:  
- One `Filter` struct that binds cleanly from query params (`?page=1&per_page=20&keyword=john&sort_by=name&sort_direction=desc`).  
- Automatic defaults, validation, and helpers (`GetLimit`, `GetOffset`, `HasKeyword`, etc.).  
- `Paging` metadata wrapper for consistent frontend-friendly responses.  
- Safe, dynamic SQL generation using `pgx` + `sqlx` (no SQL injection).  
- Column whitelisting for sorting to prevent injection.  

Combined with the existing `db` skill, every list operation becomes:
- Context-aware & transaction-safe  
- Observable (logged via slog)  
- Error-safe (wrapped via `SafeError`)  
- Production-ready (offset pagination + total count)

## When to use this skill
- Any `GET /users`, `GET /orders`, `GET /products` (or admin data tables).
- You need pagination, global search, sorting, and basic filters.
- You want consistent response shape: `{ data: [...], paging: {...} }`.
- You are replacing manual `LIMIT/OFFSET` or query string parsing in handlers.

**Never** parse query params manually in handlers or write raw `LIMIT`/`OFFSET` without this Filter.

## Project Structure (recommended)

```
internal/common/model/
├── filter.go      ← Filter + Paging (copy + enhance from ethos-go)
└── response.go    ← optional: generic ListResponse[T]
```

## Core Code (filter.go – enhanced from reference)

```go
package model

import (
    "errors"
    "net/http"
    "strconv"
    "strings"
    "time"
)

// Filter represents all query parameters for data tables / list endpoints
type Filter struct {
    // Pagination
    CurrentPage int `json:"current_page" form:"current_page" query:"current_page"`
    PerPage     int `json:"per_page" form:"per_page" query:"per_page"`

    // Global search
    Keyword string `json:"keyword" form:"keyword" query:"keyword"`

    // Sorting
    SortBy        string `json:"sort_by" form:"sort_by" query:"sort_by"`
    SortDirection string `json:"sort_direction" form:"sort_direction" query:"sort_direction"`

    // Common filters (extendable)
    StartDate *time.Time `json:"start_date,omitempty" form:"start_date" query:"start_date"`
    EndDate   *time.Time `json:"end_date,omitempty" form:"end_date" query:"end_date"`
    IsActive  *bool      `json:"is_active,omitempty" form:"is_active" query:"is_active"`
}

const (
    AscDirection       = "asc"
    DescDirection      = "desc"
    DefaultPageLimit   = 20
    DefaultCurrentPage = 1
    UnlimitedPage      = -1 // for "all" mode (use carefully)
)

func NewFilter() Filter {
    return Filter{
        CurrentPage:   DefaultCurrentPage,
        PerPage:       DefaultPageLimit,
        SortDirection: AscDirection,
    }
}

func (f *Filter) GetLimit() int {
    if f.PerPage == UnlimitedPage {
        return UnlimitedPage
    }
    return f.PerPage
}

func (f *Filter) GetOffset() int {
    if f.PerPage == UnlimitedPage {
        return 0
    }
    return (f.CurrentPage - 1) * f.PerPage
}

func (f *Filter) HasKeyword() bool { return strings.TrimSpace(f.Keyword) != "" }
func (f *Filter) HasSort() bool    { return strings.TrimSpace(f.SortBy) != "" }
func (f *Filter) IsDesc() bool     { return strings.EqualFold(f.SortDirection, DescDirection) }
func (f *Filter) IsUnlimited() bool { return f.PerPage == UnlimitedPage }

// Validate sets safe defaults and sanitizes values
func (f *Filter) Validate() {
    if f.CurrentPage < 1 {
        f.CurrentPage = DefaultCurrentPage
    }
    if f.PerPage < 1 && f.PerPage != UnlimitedPage {
        f.PerPage = DefaultPageLimit
    }
    if f.SortDirection != AscDirection && f.SortDirection != DescDirection {
        f.SortDirection = AscDirection
    }
    f.Keyword = strings.TrimSpace(f.Keyword)
}

// FilterFromRequest parses query params (supports aliases: page/per_page, search/q, order_by/order)
func FilterFromRequest(r *http.Request) Filter {
    q := r.URL.Query()
    f := NewFilter()

    // Pagination
    if v := q.Get("page"); v != "" {
        if i, err := strconv.Atoi(v); err == nil && i > 0 {
            f.CurrentPage = i
        }
    }
    if v := q.Get("per_page"); v != "" {
        if i, err := strconv.Atoi(v); err == nil && i > 0 {
            f.PerPage = i
        }
    }
    if v := q.Get("limit"); v != "" { // alias
        if i, err := strconv.Atoi(v); err == nil && i > 0 {
            f.PerPage = i
        }
    }

    // Search (multiple aliases)
    f.Keyword = strings.TrimSpace(q.Get("keyword"))
    if f.Keyword == "" {
        f.Keyword = strings.TrimSpace(q.Get("search"))
    }
    if f.Keyword == "" {
        f.Keyword = strings.TrimSpace(q.Get("q"))
    }

    // Sorting
    f.SortBy = strings.TrimSpace(q.Get("sort_by"))
    if f.SortBy == "" {
        f.SortBy = strings.TrimSpace(q.Get("order_by"))
    }
    f.SortDirection = strings.ToLower(strings.TrimSpace(q.Get("sort_direction")))
    if f.SortDirection == "" {
        f.SortDirection = strings.ToLower(strings.TrimSpace(q.Get("order")))
    }

    // Date range
    if v := q.Get("start_date"); v != "" {
        if t, err := time.Parse("2006-01-02", v); err == nil {
            f.StartDate = &t
        }
    }
    if v := q.Get("end_date"); v != "" {
        if t, err := time.Parse("2006-01-02", v); err == nil {
            f.EndDate = &t
        }
    }

    // Status
    if v := q.Get("active"); v != "" {
        b := v == "true" || v == "1"
        f.IsActive = &b
    }

    f.Validate()
    return f
}

// Paging metadata (consistent response shape)
type Paging struct {
    HasPreviousPage        bool `json:"has_previous_page"`
    HasNextPage            bool `json:"has_next_page"`
    CurrentPage            int  `json:"current_page"`
    PerPage                int  `json:"per_page"`
    TotalData              int  `json:"total_data"`
    TotalDataInCurrentPage int  `json:"total_data_in_current_page"`
    LastPage               int  `json:"last_page"`
    From                   int  `json:"from"`
    To                     int  `json:"to"`
}

var ErrInvalidPaging = errors.New("per_page must be > 0 and offset must not be negative")

func NewPaging(currentPage, perPage, totalData int) (*Paging, error) {
    if isUnlimitedPage(perPage) {
        return &Paging{
            CurrentPage:            currentPage,
            PerPage:                perPage,
            TotalData:              totalData,
            LastPage:               1,
            From:                   1,
            To:                     totalData,
            TotalDataInCurrentPage: totalData,
        }, nil
    }

    if totalData == 0 {
        return &Paging{CurrentPage: 1, PerPage: perPage, TotalData: 0, LastPage: 1}, nil
    }

    offset := (currentPage - 1) * perPage
    if perPage <= 0 || offset < 0 {
        return nil, ErrInvalidPaging
    }

    lastPage := totalData / perPage
    if totalData%perPage != 0 {
        lastPage++
    }

    to := min(offset+perPage, totalData)
    from := 0
    if to > offset {
        from = offset + 1
    }

    return &Paging{
        HasPreviousPage:        currentPage > 1,
        HasNextPage:            currentPage < lastPage,
        CurrentPage:            currentPage,
        PerPage:                perPage,
        TotalData:              totalData,
        LastPage:               lastPage,
        From:                   from,
        To:                     to,
        TotalDataInCurrentPage: to - offset,
    }, nil
}

func isUnlimitedPage(p int) bool { return p == UnlimitedPage }
func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}
```

## Usage in Repository (with pgx + sqlx)

Add this to your repository (example: `internal/repository/user_repo.go`):

```go
type UserRepo struct {
    db *db.DB
    // Optional: whitelist allowed sort columns
    allowedSortColumns map[string]bool
}

func NewUserRepo(db *db.DB) *UserRepo {
    return &UserRepo{
        db: db,
        allowedSortColumns: map[string]bool{
            "id": true, "name": true, "email": true, "created_at": true,
        },
    }
}

func (r *UserRepo) List(ctx context.Context, f model.Filter) ([]User, *model.Paging, error) {
    f.Validate()

    // 1. Count query (for Paging.TotalData)
    countQuery := `SELECT COUNT(*) FROM users WHERE 1=1`
    countArgs := []interface{}{}
    argIndex := 1

    // Build dynamic WHERE + args (safe & parameterized)
    if f.HasKeyword() {
        countQuery += ` AND (name ILIKE $` + strconv.Itoa(argIndex) + ` OR email ILIKE $` + strconv.Itoa(argIndex) + `)`
        countArgs = append(countArgs, "%"+f.Keyword+"%")
        argIndex++
    }
    if f.StartDate != nil {
        countQuery += ` AND created_at >= $` + strconv.Itoa(argIndex)
        countArgs = append(countArgs, f.StartDate)
        argIndex++
    }
    // ... add more filters similarly

    var total int
    err := r.db.SQLX.GetContext(ctx, &total, countQuery, countArgs...)
    if err != nil {
        return nil, nil, errors.New("DB_COUNT_ERROR", "Failed to count records", err, nil)
    }

    // 2. Data query
    query := `SELECT * FROM users WHERE 1=1`
    args := countArgs // reuse the same args

    // Sorting (whitelisted)
    if f.HasSort() && r.allowedSortColumns[f.SortBy] {
        dir := "ASC"
        if f.IsDesc() {
            dir = "DESC"
        }
        query += ` ORDER BY ` + f.SortBy + ` ` + dir
    } else {
        query += ` ORDER BY created_at DESC` // default
    }

    // Pagination
    limit := f.GetLimit()
    if limit != UnlimitedPage {
        query += ` LIMIT $` + strconv.Itoa(argIndex) + ` OFFSET $` + strconv.Itoa(argIndex+1)
        args = append(args, limit, f.GetOffset())
    }

    var users []User
    err = r.db.SQLX.SelectContext(ctx, &users, query, args...)
    if err != nil {
        return nil, nil, errors.New("DB_LIST_ERROR", "Failed to fetch records", err, nil)
    }

    paging, err := model.NewPaging(f.CurrentPage, f.PerPage, total)
    if err != nil {
        return nil, nil, err
    }

    slog.InfoContext(ctx, "data table query executed",
        slog.Int("total", total),
        slog.Int("returned", len(users)),
        slog.String("keyword", f.Keyword),
    )

    return users, paging, nil
}
```

## Handler Usage (with problem & logging)

```go
func ListUsers(w http.ResponseWriter, r *http.Request) {
    filter := model.FilterFromRequest(r)

    users, paging, err := userRepo.List(r.Context(), filter)
    if err != nil {
        p := errors.ToProblem(err, r.URL.Path)
        p.Write(w)
        return
    }

    response := map[string]any{
        "data":   users,
        "paging": paging,
    }
    // json response...
}
```

## Best Practices & Decision Tree

**Always**:
- Use `FilterFromRequest` + `f.Validate()` in every list handler.
- Whitelist `SortBy` columns in the repo.
- Run a separate COUNT query for accurate `Paging`.
- Return domain models + `*model.Paging`.
- Log the query summary via slog.

**Decision Tree**:
1. Simple list? → `FilterFromRequest` + repo.List.
2. Need more filters? → Extend `Filter` struct + add conditions in repo.
3. Performance critical / huge tables? → Later upgrade to cursor pagination (add `cursor` field).
4. Testing? → Pass a manual `model.Filter{...}`.

**Never**:
- Parse query params manually.
- Allow arbitrary `ORDER BY` from user input.
- Skip the COUNT query (frontend needs total/last_page).

## Integration Notes
- Put `filter.go` in `internal/common/model`.
- Combine with the `database-access-with-pgx-and-sqlx` skill (`GetQuerier` works perfectly inside `RunInTx`).
- Combine with `secure-idiomatic-error-handling` (all DB errors → SafeError).
- Combine with `structured-logging-with-slog` (request_id auto-included).
- For frontend: this matches common table libraries (TanStack Table, AG-Grid, etc.).

Follow this skill and every data table in your Go backends will be:
- Consistent
- Secure
- Performant
- Frontend-friendly

When in doubt, copy the `FilterFromRequest` + repo `List` pattern above — this is the exact evolved design used across all SemmiDev backend projects.
