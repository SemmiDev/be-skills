---
name: designing-go-http-services
description: Production-grade technique for designing and structuring Go HTTP services after years of experience (inspired by Grafana's Mat Ryer's 13+ years of lessons). Emphasizes a single `http.Handler` per service, centralized route definition, explicit dependency injection via constructor functions, middleware chaining, and avoidance of "fat handler structs". Combines seamlessly with your existing Clean Architecture, Chi router (or stdlib ServeMux), rich domain models, and all previous skills. This is the "how to organize the glue" skill that ties everything together into maintainable, testable, and observable services.
---

# Designing Go HTTP Services (Mature Idiomatic Pattern)

**Skill purpose**  
This skill captures battle-tested principles from Mat Ryer (Grafana) after 13+ years building HTTP services in Go, combined with modern 2026 Go idioms and your SemmiDev Clean Architecture.  

Core philosophy:
- One primary `http.Handler` per service (usually built in `NewServer`).
- **Explicit dependency injection** via function arguments (no hidden state in structs).
- **Centralized routes** in one place (`routes.go` or inside `router.go`).
- **Middleware chaining** applied in `NewServer` for cross-cutting concerns.
- Handlers remain thin: they parse requests, call application services, and return responses (no business logic).
- Maximize **testability** and **readability**.

This pattern works beautifully with Chi (your current router) or even plain `net/http.ServeMux`.

## Key Principles (from Grafana article + 2026 consensus)

1. **Avoid "Server Struct" Anti-Pattern**  
   Do **not** make handlers methods on a big `Server` struct. It hides dependencies and makes testing painful.

2. **NewServer Constructor Pattern**  
   Central place to wire everything:
   - Take dependencies explicitly (vertical list for readability).
   - Build the mux/router.
   - Register routes.
   - Apply middleware chain.
   - Return a single `http.Handler`.

3. **Routes in One Place**  
   All endpoints visible at a glance — huge win for maintenance.

4. **Thin Handlers**  
   Handlers only:
   - Extract params / decode JSON.
   - Call application service.
   - Handle response or error (using your `problem` skill).

5. **Middleware as Layers**  
   Apply in correct order inside `NewServer`.

## Recommended Project Structure Integration

Your existing Clean Architecture fits perfectly:

```
internal/
├── presentation/
│   └── http/
│       ├── server.go          ← NewServer + middleware chain
│       ├── routes.go          ← All route registration (centralized)
│       ├── handlers/          ← Thin handlers (one file per domain)
│       └── middleware/        ← Your existing middleware
├── application/               ← Services called by handlers
├── domain/                    ← Rich models
└── infrastructure/            ← Repos, Redis, etc.
```

## Core Code Examples

### 1. `internal/presentation/http/server.go` (NewServer – the heart)

```go
package http

import (
    "net/http"

    "yourproject/internal/application"
    "yourproject/internal/config"
    "yourproject/internal/presentation/http/handlers"
    "yourproject/internal/presentation/http/middleware"
    "log/slog"
)

func NewServer(
    cfg *config.AppConfig,
    logger *slog.Logger,
    userService *application.UserService,
    orderService *application.OrderService,
    // ... add other services vertically
) http.Handler {

    mux := chi.NewRouter() // or http.NewServeMux()

    // Register all routes in one place
    handlers.RegisterRoutes(mux, userService, orderService, logger)

    // Middleware chain (order matters)
    var h http.Handler = mux

    h = middleware.LoggingMiddleware(logger)(h)      // your slog middleware
    h = middleware.RecoveryMiddleware()(h)           // uses your ToProblem
    h = middleware.AuthMiddleware()(h)               // optional user in context
    h = middleware.TimeoutMiddleware(30 * time.Second)(h)

    return h
}
```

### 2. Centralized Routes (`internal/presentation/http/routes.go` or inside handlers)

```go
package handlers

import (
    "net/http"

    "github.com/go-chi/chi/v5"
    "yourproject/internal/application"
)

func RegisterRoutes(
    r chi.Router,
    userSvc *application.UserService,
    orderSvc *application.OrderService,
    logger *slog.Logger,
) {
    r.Route("/api/v1", func(r chi.Router) {
        // Users
        r.Route("/users", func(r chi.Router) {
            r.Post("/", makeCreateUserHandler(userSvc))
            r.Get("/{id}", makeGetUserHandler(userSvc))
        })

        // Orders
        r.Route("/orders", func(r chi.Router) {
            r.Post("/", makeCreateOrderHandler(orderSvc))
        })
    })

    // Health check
    r.Get("/health", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
    })
}
```

### 3. Thin Handler Example

```go
func makeCreateUserHandler(svc *application.UserService) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        var req dto.CreateUserRequest
        if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
            problem.New(problem.BadRequest).Write(w)
            return
        }

        if validationErrs := validator.ValidateAndGetErrors(&req); validationErrs != nil {
            problem.New(problem.UnprocessableEntity,
                problem.WithExtension("field_errors", validationErrs.ToKV()),
            ).Write(w)
            return
        }

        user, err := svc.CreateUser(r.Context(), req)
        if err != nil {
            p := errors.ToProblem(err, r.URL.Path)
            p.Write(w)
            return
        }

        respondJSON(w, http.StatusCreated, user)
    }
}
```

**Helper** (in same package):

```go
func respondJSON(w http.ResponseWriter, status int, data any) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}
```

## Best Practices Summary (Grafana + SemmiDev)

**Do**:
- Use `NewServer(...)` as the single wiring point.
- Keep route definitions centralized.
- Pass dependencies explicitly (vertical formatting for long lists).
- Make handlers thin and pure functions.
- Apply middleware by wrapping the handler.
- Use your existing skills (problem responses, slog middleware, validation, etc.).

**Avoid**:
- Fat server structs with handler methods.
- Scattered route registration.
- Implicit dependencies (global vars, context-only injection for everything).

## Integration with Your Existing Skills

- **Clean Architecture**: Handlers live in `presentation/http/`, call `application/` services.
- **Chi Router**: Already used — just wrap in `NewServer`.
- **Graceful Shutdown**: `http.Server{Handler: NewServer(...)}` + your shutdown pattern.
- **Config**: Passed to `NewServer`.
- **Logging & Errors**: Middleware + `ToProblem`.
- **Swagger**: Mount inside `NewServer` or after.

## Makefile / Build Tip

```makefile
run:
	go run cmd/server/main.go
```

In `cmd/server/main.go`:

```go
cfg := config.Load()
logger := logging.NewLogger(...)
srv := http.NewServer(cfg, logger, userSvc, orderSvc /* ... */)

httpServer := &http.Server{
    Addr:    ":" + cfg.Port,
    Handler: srv,
}
// Then your graceful shutdown pattern
```

Follow this skill and your Go HTTP services will be:
- Extremely maintainable (routes in one place)
- Highly testable (explicit deps)
- Clean and idiomatic
- Aligned with Grafana's hard-won lessons and modern Go practices

When in doubt, make `NewServer` the single source of truth for wiring your service. This is the mature way to design Go HTTP services in 2026.
