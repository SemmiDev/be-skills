---
name: chi-router-and-middleware-best-practices
description: Production-grade Chi (v5) router + middleware composition for Go backend projects. Follows 2026 best practices from the official Chi documentation, SemmiDev atstex-lab patterns, and community consensus (RequestID → RealIP → Logging → Recoverer → Timeout → custom middleware). Provides a clean, modular router setup with route grouping, context propagation, and seamless integration with all previous skills (structured-logging-with-slog, secure-idiomatic-error-handling, http-error-handling-with-problem, validation, etc.). Always use this instead of raw net/http.ServeMux or other routers.
---

# Chi Router & Middleware Best Practices

**Skill purpose**  
This skill defines the **official SemmiDev pattern** for building HTTP APIs with `github.com/go-chi/chi/v5`.  

Chi is the lightweight, idiomatic, zero-dependency router chosen across all SemmiDev backends (including atstex-lab). It stays close to `net/http`, excels at composable middleware, route grouping, and context propagation — making it perfect for clean architecture and large services.

This skill gives you:
- The exact recommended middleware stack order (critical for correctness).
- Modular route registration (per-domain or per-module).
- Custom middleware that integrates with your existing skills (slog logging, SafeError → Problem, validation, etc.).
- Context best practices for request_id, user, trace_id, etc.
- Production-ready setup with health checks, CORS, auth, timeouts, and recovery.

## When to use this skill
- Any HTTP API (REST, JSON, GraphQL gateway, etc.).
- You need clean route organization with groups and sub-routers.
- You want reusable, testable middleware.
- You are replacing manual `http.HandleFunc` chains or other routers.

**Never** register routes or middleware directly in `main.go` or mix concerns.

## Installation

```bash
go get github.com/go-chi/chi/v5
go get github.com/go-chi/chi/v5/middleware  # official middleware
```

(Chi v5 is the current stable version as of 2026.)

## Project Structure (recommended – matches SemmiDev style)

```
internal/server/
├── router.go          ← Chi router initialization + middleware stack
├── middleware/        ← Custom middleware (auth, cors, etc.)
│   ├── logging.go     ← (wraps your slog middleware)
│   ├── recovery.go    ← (uses ToProblem + SafeError)
│   ├── cors.go
│   ├── auth.go
│   └── ...
├── handlers/          ← Per-domain handlers
│   ├── user.go
│   └── ...
└── routes/            ← Optional: route registration per module
```

## Core Concepts

1. **Middleware Order Matters** (2026 consensus from Chi docs + community)
   - Early: RequestID, RealIP, CORS (if needed)
   - Then: Your structured logging (slog)
   - Then: Recovery (catches panics)
   - Then: Timeout
   - Then: Auth / Validation / Rate limiting / etc.

2. **Route Grouping**
   - `r.Route("/api/v1", func(r chi.Router) { ... })` for versioned groups
   - `r.With(middleware).Get(...)` for endpoint-specific middleware
   - `r.Mount("/admin", adminRouter())` for sub-routers

3. **Context Propagation**
   - Always use `r.WithContext(ctx)` in middleware.
   - Store request_id, trace_id, user_id, etc. (integrates with your slog and logging skill).

## Recommended Middleware Stack (router.go)

```go
package server

import (
    "net/http"
    "time"

    "github.com/go-chi/chi/v5"
    "github.com/go-chi/chi/v5/middleware"
    
    "yourproject/internal/common/logging"   // your slog middleware
    "yourproject/internal/common/errors"    // ToProblem
    "yourproject/internal/server/middleware" // your custom ones
)

func NewRouter() *chi.Mux {
    r := chi.NewRouter()

    // === GLOBAL MIDDLEWARE STACK (order is critical) ===
    r.Use(middleware.RequestID)           // adds X-Request-ID
    r.Use(middleware.RealIP)              // handles proxies
    r.Use(middleware.GetHead)             // auto-responds to HEAD
    r.Use(middleware.NoCache)             // optional: prevent caching
    r.Use(logging.LoggingMiddleware)      // ← your structured slog middleware from previous skill
    r.Use(middleware.Recoverer)           // must come AFTER logging
    r.Use(middleware.Timeout(30 * time.Second))
    r.Use(middleware.CleanPath)           // canonicalize paths

    // Custom global middleware (CORS, etc.)
    r.Use(middleware.CORS)                // your custom CORS (see below)

    // === HEALTH CHECK (outside any auth) ===
    r.Get("/health", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        w.Write([]byte(`{"status":"ok"}`))
    })

    // === API ROUTES ===
    r.Route("/api/v1", func(r chi.Router) {
        // Public routes (no auth)
        r.Route("/public", func(r chi.Router) {
            // ...
        })

        // Protected routes
        r.Route("/", func(r chi.Router) {
            r.Use(middleware.AuthMiddleware) // your custom auth (sets user in ctx)
            // register domain routes here
            userRoutes(r)
            // ...
        })
    })

    return r
}
```

## Custom Middleware Examples (integrate with your skills)

### 1. Recovery Middleware (uses secure error handling)

```go
// internal/server/middleware/recovery.go
func RecoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if rec := recover(); rec != nil {
                err := fmt.Errorf("panic: %v", rec)
                p := errors.ToProblem(err, r.URL.Path) // from secure-error skill
                p.Write(w)
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```

### 2. Auth Middleware (sets user in context)

```go
// internal/server/middleware/auth.go
func AuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        // validate token → get user
        user, err := authService.ValidateToken(r.Context(), token)
        if err != nil {
            p := problem.New(problem.Unauthorized).Write(w) // or use ToProblem
            return
        }

        ctx := context.WithValue(r.Context(), "user", user)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

### 3. Validation Middleware (optional per-route)

Use your `Validator.ValidateAndWriteProblem` helper from the validation skill.

## Route Registration (modular – inspired by atstex-lab & clean architecture)

```go
// internal/server/routes/user.go
func UserRoutes(r chi.Router) {
    r.Get("/", getUsers)
    r.Post("/", createUser)
    r.Route("/{userID}", func(r chi.Router) {
        r.Use(userCtxMiddleware) // inject user into context
        r.Get("/", getUser)
        r.Put("/", updateUser)
        r.Delete("/", deleteUser)
    })
}

// In router.go
func userRoutes(r chi.Router) {
    UserRoutes(r) // or r.Route("/users", UserRoutes)
}
```

## Handler Example (clean & integrated)

```go
func getUser(w http.ResponseWriter, r *http.Request) {
    userID := chi.URLParam(r, "userID")
    user, err := service.GetUser(r.Context(), userID)
    if err != nil {
        p := errors.ToProblem(err, r.URL.Path)
        p.Write(w)
        return
    }
    // json response
}
```

## Best Practices & Decision Tree (2026 consensus)

**Always**:
- Use the exact middleware order above.
- Put logging **before** Recoverer.
- Use `r.Route()` and `r.With()` for grouping.
- Store only necessary data in context.
- Register routes in separate functions per domain.
- Add `/health` endpoint.
- Use `chi.URLParam` for path params.

**Decision Tree**:
1. Cross-cutting concern? → Global `r.Use(...)`
2. Domain-specific? → `r.Route(...)` + group middleware
3. Single endpoint only? → `r.With(specificMiddleware).Get(...)`
4. Sub-API? → `r.Mount("/admin", adminRouter())`
5. Need auth on some routes only? → Group with `r.Use(authMiddleware)`

**Never**:
- Register middleware after routes.
- Log sensitive data (use your slog redaction).
- Ignore context cancellation in handlers.
- Put business logic in middleware.

## Integration Notes for Your Backend Projects
- Call `server.NewRouter()` in `main.go` and pass to `http.ListenAndServe`.
- Combine with `structured-logging-with-slog` (already included).
- Combine with `secure-idiomatic-error-handling` + `http-error-handling-with-problem` (via recovery & handlers).
- Combine with validation skill (add validation middleware where needed).
- For atstex-lab-style projects: keep one `router.go` + modular route files.
- Testing: handlers are plain `http.HandlerFunc` — use `httptest` easily.
- Docs: use `chi/docgen` or OpenAPI tools on the router.

Follow this skill and every API in your Go backends will be:
- Lightweight & fast
- Secure & observable
- Modular & maintainable
- Fully integrated with your existing skills
