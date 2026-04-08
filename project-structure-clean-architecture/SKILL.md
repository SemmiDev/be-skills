---
name: project-structure-clean-architecture-idiomatic-go
Description: Canonical, production-grade project layout for SemmiDev Go backends combining **Clean Architecture** (Uncle Bob’s Dependency Rule + layered separation) with **idiomatic Go conventions** (standard project layout, `internal/`, `cmd/`, rich domain models, etc.).  

This structure is directly evolved from `atstex-lab` (the reference repo) and aligns with 2026 Go community consensus (Clean Arch + Hexagonal/Ports & Adapters hybrids used in high-scale services). It enforces:
- **Dependency Rule**: Inner layers (Domain) never depend on outer layers (Infrastructure, UI).
- **Testability**: Domain & Application are pure and unit-testable.
- **Maintainability**: Clear boundaries, no import cycles, easy swapping of DB/storage/AI.
- **Go Idioms**: `internal/` privacy, `cmd/` entrypoints, `//go:embed`, rich domain objects.

**When to use this skill**  
Use this exact layout for **every new or refactored Go backend** (API + optional server-rendered pages or React SPA).

---

### Official SemmiDev Project Tree (2026)

```bash
your-project/
├── cmd/                          # Application entrypoints (one per binary)
│   └── server/                   # Main HTTP API server (most common)
│       └── main.go               # Calls config.Load(), NewRouter(), graceful shutdown
│
├── internal/                     # Private code – NEVER imported from outside
│   ├── config/                   # Clean config loading (from previous skill)
│   │   └── config.go
│   │
│   ├── domain/                   # Rich Domain Model (DDD tactical patterns)
│   │   ├── user/                 # One bounded context / aggregate per sub-package
│   │   │   ├── aggregate.go      # User (Aggregate Root)
│   │   │   ├── entity.go
│   │   │   ├── valueobject.go    # Email, Money, etc.
│   │   │   ├── factory.go
│   │   │   └── event.go          # Domain events
│   │   └── order/                # Another aggregate
│   │
│   ├── application/              # Use Cases / Application Services (orchestrate domain)
│   │   ├── user/
│   │   │   └── service.go        # CreateUser, UpdateProfile, etc.
│   │   └── order/
│   │
│   ├── infrastructure/           # Adapters to external world
│   │   ├── database/             # pgx + sqlx + migrations
│   │   │   └── repository.go     # Implements domain.Repository interfaces
│   │   ├── storage/              # S3 abstraction
│   │   ├── email/                # SMTP sender
│   │   ├── ai/                   # OpenAI / Gemini client (example)
│   │   └── repository/           # Shared repo interfaces (if needed)
│   │
│   ├── presentation/             # Delivery layer (HTTP handlers, middleware)
│   │   ├── http/
│   │   │   ├── handlers/         # Per-domain handlers (GetUser, etc.)
│   │   │   ├── middleware/       # Auth, Logging, Recovery, Validation
│   │   │   └── router.go         # Chi router + Swagger mount
│   │   └── static.go             # Embedded React SPA or server-rendered HTML
│   │
│   ├── auth/                     # Google OAuth + session middleware (exact atstex-lab style)
│   ├── common/                   # Cross-cutting helpers (validator, errors, logging, random/uuid)
│   │   ├── logging/
│   │   ├── errors/
│   │   ├── validator/
│   │   └── random/
│   │
│   └── server/                   # Server setup glue (NewRouter, InitTemplates, etc.)
│
├── web/                          # Embedded frontend assets (exact atstex-lab)
│   ├── embed.go                  # //go:embed templates + static
│   ├── templates/                # layouts/, partials/, pages/
│   └── static/                   # css/, js/, img/ (or React dist/)
│
├── db/
│   └── migration/                # golang-migrate SQL files (timestamped)
│
├── docs/                         # Generated Swagger (swag init)
│   ├── docs.go
│   └── swagger.json
│
├── .github/
│   └── workflows/                # CI/CD (lint, test, build, deploy)
│
├── deploy/                       # Ansible / Terraform / Helm (as in atstex-lab)
│
├── .env.example
├── .dockerignore
├── .golangci.yml
├── Dockerfile                    # Multi-stage (from previous skill)
├── Makefile                      # build, migrate, swag, docker-build, etc.
├── compose.yml                   # Local dev with Postgres, etc.
├── go.mod
├── go.sum
├── README.md
├── SPEC.md                       # Technical specification (as in atstex-lab)
└── LICENSE
```

### Layer-by-Layer Explanation (Clean Architecture + Go Idioms)

| Layer                  | Folder                          | Responsibility                              | Depends On          | Imports From          |
|------------------------|---------------------------------|---------------------------------------------|---------------------|-----------------------|
| **Domain**             | `internal/domain/`             | Business rules, entities, aggregates, invariants | Nothing             | —                     |
| **Application**        | `internal/application/`        | Use cases / business workflows              | Domain              | domain/               |
| **Interface/Adapters** | `internal/presentation/`       | HTTP handlers, middleware, routers, Swagger | Application         | application/, domain/ |
| **Infrastructure**     | `internal/infrastructure/`     | DB, S3, Email, AI clients, Repositories    | Domain + Application| domain/ (ports)       |
| **Cross-cutting**      | `internal/common/`             | Logging, errors, validator, UUID, etc.      | —                   | —                     |
| **Delivery**           | `cmd/server/` + `internal/server/` | Main + router wiring                        | All above           | internal/             |
| **External**           | `web/`, `db/migration/`        | Assets & schema                             | —                   | —                     |

**Dependency Rule (Clean Arch)**: Arrows only point inward. Infrastructure depends on Domain interfaces (ports), never the other way around.

### Why This Structure Wins (2026 Consensus)

- Matches **atstex-lab** exactly (cmd/server, internal/auth/config, web/, db/migration, docs/).
- Follows **official Go project layout** (`cmd/`, `internal/`) – no `pkg/` unless truly shared.
- Combines **Clean Architecture** (layers + dependency rule) with **Hexagonal/Ports & Adapters** (repositories as interfaces in domain).
- Rich Domain in `domain/` (from previous skill) – no anemic models.
- Easy testing: domain & application are pure; infrastructure is mocked.
- CI/CD friendly, Docker friendly, Kubernetes friendly.
- Scales to microservices (each bounded context can become its own repo later).

### Recommended Makefile Targets (from atstex-lab + skills)

```makefile
migrate-up, migrate-create, swag, build-frontend, docker-build, test, lint, run-dev
```

### Migration / Evolution Path

1. Start with this structure for new projects.
2. For very simple CRUD → you can collapse `application/` + `presentation/` initially.
3. For large monoliths → split `internal/domain/` into multiple bounded contexts.
4. When extracting microservices → each service keeps the same internal layout.

**Follow this skill and every SemmiDev Go project will have the same clean, consistent, maintainable, and idiomatic structure** — exactly as evolved from atstex-lab and aligned with 2026 Go + Clean Architecture best practices.

When in doubt, copy the tree above and place each previous skill into its corresponding folder. This is the final foundational skill that ties everything together.
