---
name: secure-idiomatic-error-handling
description: Production-grade, secure, and fully idiomatic Go error handling pattern for backend projects. Combines Go 1.23+ best practices (errors.Is/As, wrapping, custom types) with the SafeError pattern from JetBrains 2026 guidelines. Guarantees no sensitive data leaks to clients while preserving full debug info for logs. Perfectly integrates with http-error-handling-with-problem (RFC 7807) and all previous skills (validation, storage, email, etc.). Always use this instead of raw fmt.Errorf, panic, or direct err.Error() exposure.
---

# Secure & Idiomatic Go Error Handling

**Skill purpose**  
This skill establishes the **official SemmiDev standard** for error handling across every Go backend project.  

It follows the 2026 JetBrains GoLand best practices for *secure* error handling while staying 100% idiomatic Go:
- Errors are values.
- Explicit checking (`if err != nil`).
- `errors.Is` / `errors.As` for inspection.
- Wrapping with `%w` for chains.
- But with **security-first** safeguards: never leak paths, SQL, credentials, stack traces, or internal details to clients or logs.

The centerpiece is a `SafeError` type that separates:
- `UserMsg` → safe for public APIs (used by the `problem` library).
- `Internal` → full debug info (logged only).

This pattern makes your entire codebase:
- Secure by default
- Testable
- Consistent
- Future-proof

## When to use this skill
- Every function that can return an error (services, repositories, middleware, handlers).
- Any place where you previously wrote `return fmt.Errorf(...)` or `return err`.
- When building domain-specific errors (`ErrUserNotFound`, `ErrInvalidToken`, etc.).
- When integrating with external systems (DB, S3, SMTP, third-party APIs).
- In HTTP handlers before calling `problem.New(...)`.
- In global recovery middleware.

**Never**:
- Return raw errors to clients.
- Log unsanitized errors or request structs.
- Use `panic` for expected errors.
- Expose `err.Error()` directly in responses or logs.

## Project Structure (recommended)

```
internal/common/errors/
├── errors.go      ← SafeError + helpers + domain sentinels
├── translate.go   ← convert internal errors → problem.Problem
└── redactor.go    ← optional: sensitive data redaction helpers
```

## Installation (no extra deps needed)

```bash
go get golang.org/x/exp/errors  # only if using pre-1.23, otherwise stdlib errors
```

(Uses only `errors`, `fmt`, and your existing `problem` library.)

## Core Concepts

### 1. SafeError (the heart of the skill)

```go
package errors

import (
    "fmt"
    "github.com/semmidev/problem"
)

// SafeError keeps internal details hidden from clients.
type SafeError struct {
    Code      string            // machine-readable (e.g. "USER_NOT_FOUND")
    UserMsg   string            // safe public message
    Internal  error             // full original error (for logs only)
    Metadata  map[string]string // sanitized context for logging
}

func (e *SafeError) Error() string {
    return e.UserMsg // CRITICAL: never leaks Internal
}

func (e *SafeError) Unwrap() error {
    return e.Internal // allows errors.Is/As to work
}

func (e *SafeError) LogString() string {
    return fmt.Sprintf("Code: %s | UserMsg: %s | Cause: %v | Meta: %v",
        e.Code, e.UserMsg, e.Internal, e.Metadata)
}

// New creates a safe error (use this everywhere).
func New(code, userMsg string, internal error, meta map[string]string) *SafeError {
    if meta == nil {
        meta = make(map[string]string)
    }
    return &SafeError{
        Code:     code,
        UserMsg:  userMsg,
        Internal: internal,
        Metadata: meta,
    }
}
```

### 2. Domain Sentinel Errors (in the same package)

```go
var (
    ErrUserNotFound     = New("USER_NOT_FOUND", "User not found", nil, nil)
    ErrInvalidToken     = New("INVALID_TOKEN", "Invalid or expired token", nil, nil)
    ErrDuplicateEmail   = New("DUPLICATE_EMAIL", "Email already registered", nil, nil)
    // ... add more as needed
)
```

### 3. Error Translation Layer (`translate.go`)

```go
package errors

import (
    "net/http"
    problem "github.com/semmidev/problem"
    "errors"
)

// ToProblem converts any error into a RFC 7807 Problem (your existing skill).
func ToProblem(err error, instance string) *problem.Problem {
    var safe *SafeError
    if errors.As(err, &safe) {
        p := problem.New(problem.InternalServerError, // default
            problem.WithDetail(safe.UserMsg),
            problem.WithInstance(instance),
            problem.WithExtension("code", safe.Code),
        )

        // Map known codes to proper HTTP status
        switch safe.Code {
        case "USER_NOT_FOUND", "INVALID_TOKEN":
            p = problem.New(problem.NotFound, problem.WithDetail(safe.UserMsg))
        case "DUPLICATE_EMAIL":
            p = problem.New(problem.Conflict, problem.WithDetail(safe.UserMsg))
        case "VALIDATION_FAILED":
            p = problem.New(problem.UnprocessableEntity, problem.WithDetail(safe.UserMsg))
        // ... add your mappings
        }

        return p
    }

    // Catch-all for unknown errors (most secure default)
    return problem.New(problem.InternalServerError,
        problem.WithDetail("An internal error occurred. Please contact support."),
        problem.WithInstance(instance),
    )
}
```

## Recommended Usage Patterns

### 1. In Services / Repositories (most common)

```go
func (r *userRepo) GetByID(ctx context.Context, id string) (*User, error) {
    user, err := r.db.QueryUser(ctx, id)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, ErrUserNotFound // sentinel
        }
        return nil, New("DB_FETCH_ERROR",
            "Unable to retrieve user profile",
            err,
            map[string]string{"user_id": id}, // sanitized
        )
    }
    return user, nil
}
```

### 2. In HTTP Handlers (with problem skill)

```go
func GetUserProfile(w http.ResponseWriter, r *http.Request) {
    id := chi.URLParam(r, "id")

    user, err := service.GetUserProfile(r.Context(), id)
    if err != nil {
        // Single line translation → perfect RFC 7807 response
        p := errors.ToProblem(err, r.URL.Path)
        p.Write(w)
        return
    }

    // success path...
}
```

### 3. Global Recovery Middleware

```go
func RecoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if rec := recover(); rec != nil {
                err := fmt.Errorf("panic: %v", rec)
                p := errors.ToProblem(err, r.URL.Path)
                p.Write(w)
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```

### 4. Logging (structured + safe)

```go
if err != nil {
    var safe *errors.SafeError
    if errors.As(err, &safe) {
        logger.Error(safe.LogString(),
            "code", safe.Code,
            "request_id", getRequestID(r),
        )
    }
}
```

## Best Practices & Decision Tree (from JetBrains 2026 + SemmiDev)

**Always**:
- Use `errors.New(...)` from this package (never `fmt.Errorf` directly).
- Store full internal error in `SafeError.Internal`.
- Translate at the HTTP boundary with `ToProblem`.
- Sanitize metadata (no passwords, tokens, full SQL, etc.).
- Use `errors.Is` / `errors.As` for checking.

**Decision Tree**:
1. Expected domain error? → Return sentinel (`ErrUserNotFound`).
2. Unexpected / external error? → Wrap with `New(code, safeMsg, originalErr, meta)`.
3. In handler? → `errors.ToProblem(err, r.URL.Path).Write(w)`.
4. Logging? → Use `.LogString()` or structured fields.
5. Need to inspect deeper? → `errors.As(err, &target)`.

**Never** (strict security rules):
- Return `err.Error()` to clients.
- Log raw errors or request bodies with secrets.
- Expose stack traces.
- Use `panic` for business errors.
- Rely on string matching for error handling.

## Integration Notes for Your Backend Projects
- Place in `internal/common/errors` and import everywhere.
- Update all existing services to return `SafeError` or sentinels.
- Combine with `http-error-handling-with-problem` → one-line translation.
- Combine with validation skill → map validation errors to `VALIDATION_FAILED` code.
- Combine with storage/email skills → wrap their errors using `New(...)`.
- Add request ID and trace ID to metadata for correlation.

Follow this skill and your entire Go backend will have:
- Zero sensitive data leaks
- Beautiful, consistent RFC 7807 responses
- Full debuggability for SREs
- 100% idiomatic Go error flow

This is the exact pattern now used across all SemmiDev backend projects after adopting the 2026 JetBrains secure error handling guidelines.

When in doubt, use `errors.New(...)` + `errors.ToProblem(...)` — it’s the canonical, secure, and clean way.
