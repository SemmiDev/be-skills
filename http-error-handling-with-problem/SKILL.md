```markdown
---
name: http-error-handling-with-problem
description: Standardizes HTTP error responses in Go backend APIs using github.com/semmidev/problem (full RFC 7807 Problem Details implementation). Use this skill for any HTTP handler that needs to return consistent, machine-readable errors (4xx/5xx) with optional custom extensions, error wrapping, and direct ResponseWriter integration. Always prefer this over manual JSON error responses or plain http.Error.
---

# HTTP Error Handling with Problem Library

**Skill purpose**  
This skill provides the complete, idiomatic way to handle and return errors in Go HTTP backends using the `github.com/semmidev/problem` library. It guarantees RFC 7807 compliance, zero-dependency consistency across all your microservices, and seamless integration with any router (`net/http`, Chi, Gin, Echo, etc.).

## When to use this skill
- You are writing or reviewing any HTTP handler that can return an error (404, 400, 422, 500, etc.).
- You need to propagate domain/validation/database errors to the client in a standardized JSON format.
- You want to add custom context (e.g. `trace_id`, `balance`, `field_errors`) without breaking the RFC 7807 contract.
- You are replacing legacy `http.Error(w, msg, code)` or manual `json` marshaling of error structs.
- You are implementing error middleware or a global recovery handler.
- You need to wrap lower-level errors while preserving the original error for logging/debugging (`errors.Is` / `errors.As` support).

**Never use** plain `http.Error`, custom error structs with ad-hoc JSON, or non-RFC 7807 formats unless the client explicitly requires it.

## Installation
Add the library to your Go module:
```bash
go get github.com/semmidev/problem
```

Import (use the short alias for readability):
```go
import (
    "net/http"
    
    problem "github.com/semmidev/problem"
)
```

## Core Concepts (always follow)

1. **Predefined Templates** – Use the built-in ones for common HTTP statuses:
   - `problem.BadRequest`
   - `problem.Unauthorized`
   - `problem.Forbidden`
   - `problem.NotFound`
   - `problem.MethodNotAllowed`
   - `problem.UnprocessableEntity`
   - `problem.InternalServerError`
   - … (full list in the library’s `registry.go`)

2. **Custom TypeTemplate** – For domain-specific errors (strongly recommended):
   ```go
   var (
       ErrOutOfCredit = problem.TypeTemplate{
           Type:   "https://api.yourdomain.com/probs/out-of-credit",
           Title:  "You do not have enough credit.",
           Status: http.StatusForbidden,
       }
       
       ErrInvalidTaskStatus = problem.TypeTemplate{
           Type:   "https://api.yourdomain.com/probs/invalid-task-status",
           Title:  "Invalid task status transition.",
           Status: http.StatusUnprocessableEntity,
       }
   )
   ```

3. **Problem Construction** – Always use the fluent builder:
   ```go
   p := problem.New(template, options...)
   ```

4. **Writing the Response** – Call `p.Write(w)` – it sets the correct `Content-Type: application/problem+json` and status code automatically.

## Recommended Usage Patterns

### 1. Direct handler error (most common)
```go
func GetUser(w http.ResponseWriter, r *http.Request) {
    id := chi.URLParam(r, "id")
    user, err := service.GetUserByID(r.Context(), id)
    if err != nil {
        // Convert domain error to Problem
        p := problem.New(problem.NotFound,
            problem.WithDetail("User with id "+id+" was not found"),
            problem.WithInstance(r.URL.Path),
            problem.WithExtension("user_id", id),
        )
        p.Write(w)
        return
    }
    // success path...
}
```

### 2. Error wrapping (recommended for layered architecture)
```go
func queryDatabase(ctx context.Context) error {
    row, err := db.QueryRowContext(ctx, query)
    if err != nil {
        return problem.Wrap(err, problem.InternalServerError,
            problem.WithDetail("Database query failed"),
            problem.WithExtension("trace_id", getTraceID(ctx)),
        )
    }
    return nil
}

func handler(w http.ResponseWriter, r *http.Request) {
    if err := queryDatabase(r.Context()); err != nil {
        if p, ok := problem.IsProblem(err); ok {
            p.Write(w)
            return
        }
        // fallback
        problem.New(problem.InternalServerError).Write(w)
    }
}
```

### 3. Validation → 422 Unprocessable Entity (common pattern)
```go
type CreateTaskRequest struct {
    Title string `validate:"required"`
    // ...
}

func CreateTask(w http.ResponseWriter, r *http.Request) {
    var req CreateTaskRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        problem.New(problem.BadRequest,
            problem.WithDetail("Invalid JSON payload"),
        ).Write(w)
        return
    }

    if err := validator.Validate(req); err != nil {
        p := problem.New(problem.UnprocessableEntity,
            problem.WithDetail("Validation failed"),
            problem.WithExtension("field_errors", err), // map or slice
        )
        p.Write(w)
        return
    }
    // ...
}
```

### 4. Global Recovery / Middleware (Chi example)
```go
func RecoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if rec := recover(); rec != nil {
                problem.New(problem.InternalServerError,
                    problem.WithDetail("Internal server error"),
                    problem.WithExtension("panic", fmt.Sprintf("%v", rec)),
                ).Write(w)
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```

## Best Practices & Decision Tree

**Always include**:
- `WithDetail(...)` – human-readable explanation.
- `WithInstance(r.URL.Path)` – helps clients correlate the error to the request.
- At least one custom extension when it adds value (trace_id, field name, balance, etc.).

**Decision Tree**:
1. Is this a standard HTTP error? → Use predefined template.
2. Is this a domain/business error? → Define a `TypeTemplate` once and reuse.
3. Do you have a lower-level error you want to preserve? → Use `problem.Wrap`.
4. Do you need extra context? → `WithExtension` or `WithExtensions(map)`.
5. Are you in a middleware/recovery path? → Always check `problem.IsProblem(err)` first.

**Never**:
- Manually marshal JSON for errors.
- Omit the `type` field (the library always provides it).
- Use `http.StatusText` as the only information.
- Return 500 for client errors (4xx).

## Testing Tips
```go
// Example test helper
func assertProblemResponse(t *testing.T, resp *http.Response, expectedStatus int, expectedType string) {
    if resp.StatusCode != expectedStatus {
        t.Fatalf("expected status %d, got %d", expectedStatus, resp.StatusCode)
    }
    var p problem.Problem
    json.NewDecoder(resp.Body).Decode(&p)
    if p.Type != expectedType {
        t.Fatalf("expected type %s, got %s", expectedType, p.Type)
    }
}
```

## Integration Notes for Your Backend Projects
- Place all custom `TypeTemplate`s in a shared `internal/problem` or `pkg/problem` package so they are reused across services.
- Add the library to your project’s `go.mod` once and keep it updated.
- Document your custom problem types in your OpenAPI/Swagger spec (the `type` URI should be a stable link).
- Use this skill together with your logging middleware – always log the original wrapped error (`problem.IsProblem` + `errors.Unwrap`).
