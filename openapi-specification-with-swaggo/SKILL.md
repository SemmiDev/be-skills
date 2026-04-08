---
name: openapi-specification-with-swaggo
description: Production-grade OpenAPI/Swagger 2.0 documentation generation for Go Chi-based APIs using swaggo/swag + swaggo/http-swagger. Automatically generates `docs/` folder (swagger.json + docs.go) from code comments. Provides clean Chi mounting (`r.Mount("/swagger", httpSwagger.WrapHandler)`) and full integration with the existing chi-router-and-middleware-best-practices skill. Follows the exact SemmiDev pattern used in atstex-lab and all other backend projects. Always document every public endpoint with swaggo annotations instead of manual Swagger files or Postman collections.
---

# OpenAPI Specification with Swaggo

**Skill purpose**  
This skill defines the **official SemmiDev pattern** for generating and serving live OpenAPI (Swagger 2.0) documentation using `swaggo/swag` + `swaggo/http-swagger`.  

It is the exact approach used across all SemmiDev projects (including atstex-lab):  
- Zero-maintenance, comment-driven documentation.  
- `swag init` generates everything automatically.  
- Swagger UI is mounted under `/swagger` (or `/docs`) with one line of Chi code.  
- Full support for request/response models, query/path params, tags, security schemes, examples, and error responses.  
- Works perfectly with your existing Chi router, middleware stack, validation, and problem-error handling.  

Result: A beautiful, always-up-to-date interactive API explorer at `http://localhost:8080/swagger/index.html` — perfect for frontend teams, QA, and external consumers.

## When to use this skill
- Every public HTTP API endpoint (handlers in `internal/server/handlers/`).
- You want living documentation that stays in sync with code.
- You need to generate client SDKs, contract tests, or Postman collections from the same source.
- You are replacing manual Swagger YAML/JSON files or outdated docs.

**Never** write or maintain a separate `swagger.yaml` or `openapi.json` by hand.

## Installation

```bash
# 1. CLI tool (run once globally)
go install github.com/swaggo/swag/cmd/swag@latest

# 2. Runtime library + Chi support
go get github.com/swaggo/swag
go get github.com/swaggo/http-swagger
```

## Project Structure (matches SemmiDev / atstex-lab style)

```
docs/                  ← GENERATED (never edit manually)
├── docs.go
├── swagger.json
├── swagger.yaml
internal/server/
├── router.go          ← Mount Swagger UI here
├── handlers/
│   └── user.go        ← Add swaggo comments above handlers
└── ...
Makefile               ← Add `swag` target
```

## Core Workflow

### 1. Add Swaggo Comments to Handlers

Place annotations **immediately above** each handler function (or in a common `// @title` block).

**Example (`internal/server/handlers/user.go`)**

```go
package handlers

import (
    "net/http"
    "yourproject/internal/common/errors"
    "yourproject/internal/dto"
)

// @title          Atstex Lab API
// @version        1.0
// @description    Official API for Atstex Lab platform
// @host           api.atstex.com
// @BasePath       /api/v1
// @schemes        https http
// @securityDefinitions.apikey ApiKeyAuth
// @in             header
// @name           Authorization

// GetUser godoc
// @Summary      Get user by ID
// @Description  Returns a single user by UUID
// @Tags         users
// @Accept       json
// @Produce      json
// @Param        userID  path      string  true  "User UUID (v7)"
// @Success      200     {object}  dto.UserResponse
// @Failure      400     {object}  problem.Problem
// @Failure      404     {object}  problem.Problem
// @Router       /users/{userID} [get]
func GetUser(w http.ResponseWriter, r *http.Request) {
    // ... your handler code
}
```

**Common tags you should always use**:
- `@Summary`, `@Description`
- `@Tags` (group by domain)
- `@Accept`, `@Produce`
- `@Param` (path/query/body/header)
- `@Success`, `@Failure` (link to response models)
- `@Security ApiKeyAuth` (for JWT)

**Response models** (in `dto/` or `internal/dto`):

```go
// UserResponse godoc
// @Description User response model
type UserResponse struct {
    ID    string `json:"id" example:"0192..."`
    Email string `json:"email" example:"user@example.com"`
}
```

### 2. Generate Documentation

Add to your `Makefile`:

```makefile
.PHONY: swag
swag: ## Generate OpenAPI/Swagger docs
	swag init -g ./internal/server/router.go -o ./docs --parseDependency --parseInternal --parseDepth 1
	@echo "✅ Swagger docs generated in ./docs"
```

Run:
```bash
make swag
```

This creates/updates the `docs/` folder. Commit the generated files (they are part of the source).

### 3. Mount Swagger UI in Chi Router

Update `internal/server/router.go` (from your existing chi-router skill):

```go
import (
    httpSwagger "github.com/swaggo/http-swagger"
    _ "yourproject/docs" // ← IMPORTANT: import the generated docs
)

func NewRouter() *chi.Mux {
    r := chi.NewRouter()

    // ... your existing middleware stack ...

    // === SWAGGER UI (always last, after all API routes) ===
    r.Get("/swagger/*", httpSwagger.Handler(
        httpSwagger.URL("/swagger/doc.json"), // point to the generated JSON
        httpSwagger.DeepLinking(true),
        httpSwagger.DocExpansion("list"),
        httpSwagger.PersistAuthorization(true),
    ))

    // API routes (they take precedence)
    r.Route("/api/v1", func(r chi.Router) {
        // your routes
    })

    return r
}
```

**Alternative (using Mount for cleaner path)**:

```go
r.Mount("/swagger", httpSwagger.WrapHandler)
```

Access at: `http://localhost:8080/swagger/index.html`

## Best Practices & Decision Tree (2026 SemmiDev standard)

**Always**:
- Run `make swag` (or `swag init`) **before** every commit/PR that changes handlers.
- Put the main `@title` / `@version` / `@BasePath` comment in the router file or a dedicated `docs.go`.
- Use meaningful `@Tags` for logical grouping.
- Link every response to a struct with `// @Success 200 {object} dto.XXXResponse`.
- Document all error cases using your `problem.Problem` type.
- Add examples with `example:"value"` in structs.

**Decision Tree**:
1. New endpoint? → Add full swaggo comment block above handler.
2. Shared response model? → Document the struct once with godoc.
3. Auth required? → Use `@Security ApiKeyAuth` + global security definition.
4. Production? → Keep `/swagger` exposed (internal tool) or protect with middleware.
5. Want OpenAPI 3? → Use `swag init --outputType yaml` and convert, or stick with Swagger 2.0 (swaggo default).

**Never**:
- Edit `docs/` files manually.
- Forget to import `_ "yourproject/docs"`.
- Mount Swagger before API routes (it would catch API paths).
- Ship without running `swag init`.

## Integration Notes for Your Backend Projects
- Place Swagger mount **after** all API routes but **before** the static SPA handler (if using embedded React).
- Combine with `chi-router-and-middleware-best-practices` → one-line addition.
- Combine with `graceful-shutdown-with-signal-handling` → no changes needed.
- Combine with `http-error-handling-with-problem` → all errors appear correctly in Swagger.
- CI/CD: Add `make swag` to your build pipeline and fail if docs are outdated.
- For atstex-lab-style projects: `/swagger` is already mounted exactly this way.

Follow this skill and every endpoint in your Go backends will have:
- Beautiful, interactive, always-accurate documentation
- Zero extra maintenance
- Full OpenAPI compatibility

When in doubt, copy the comment style + `swag init` + Chi mount pattern above — this is the exact production pattern used across all SemmiDev backend projects (including atstex-lab).

Run `make swag` today and enjoy live Swagger UI! 🚀
