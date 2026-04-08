---
name: clean-idiomatic-config-loading
description: Minimal, zero-dependency (in production), and fully idiomatic configuration loading for Go backend projects. Directly based on the exact pattern from SemmiDev/atstex-lab/internal/config/config.go (godotenv + simple getEnv helper + AppConfig struct). Provides a single Load() function that reads .env (dev only) + system environment variables, supplies sensible defaults, and supports required fields. Perfect integration with graceful-shutdown-with-signal-handling, structured-logging-with-slog, and all other skills. Always load config once at startup in main.go instead of scattering os.Getenv calls.
---

# Clean & Idiomatic Config Loading

**Skill purpose**  
This skill establishes the **official SemmiDev pattern** for configuration loading, taken directly from `atstex-lab/internal/config/config.go` and refined with 2026 best practices.  

It is deliberately **minimal and lightweight**:
- No heavy libraries like Viper in production.
- Only `github.com/joho/godotenv` (dev-only convenience for `.env` files).
- Simple struct + `getEnv` helper pattern.
- Built-in defaults and required-field checking.
- Safe secret redaction when logging the loaded config.
- One-time load at application startup.

This keeps your `main.go` clean, your config explicit, and your binary small.

## When to use this skill
- Every Go service (`cmd/server/main.go`, workers, CLI tools).
- You need environment-driven configuration (Docker, Kubernetes, Railway, Fly.io, etc.).
- You want `.env` support during local development.
- You are replacing scattered `os.Getenv` calls or complex config libraries.

**Never** read `os.Getenv` directly in handlers, services, or middleware.

## Project Structure (exact match to atstex-lab)

```
internal/config/
└── config.go      ← All config lives here
```

## Canonical Implementation (`internal/config/config.go`)

```go
package config

import (
	"fmt"
	"log"
	"os"
	"strconv"

	"github.com/joho/godotenv"
)

// AppConfig holds all application configuration.
// Keep fields flat and well-named. Add new fields here as needed.
type AppConfig struct {
	// Server
	Port string `env:"PORT"`

	// Database
	DatabaseURL string `env:"DATABASE_URL"`

	// Authentication / OAuth
	GoogleClientID     string `env:"GOOGLE_CLIENT_ID"`
	GoogleClientSecret string `env:"GOOGLE_CLIENT_SECRET"`
	GoogleCallbackURL  string `env:"GOOGLE_CALLBACK_URL"`

	// AI / LLM settings (example from atstex-lab)
	AIProvider string `env:"AI_PROVIDER"` // "openai", "anthropic", "ollama", etc.
	AIModel    string `env:"AI_MODEL"`
	AIAPIKey   string `env:"AI_API_KEY"`   // falls back to OPENAI_API_KEY if empty
	AIBaseURL  string `env:"AI_BASE_URL"`  // optional for custom providers

	// Add your own fields here...
	// Example:
	// JWTSecret          string `env:"JWT_SECRET"`
	// S3Bucket           string `env:"S3_BUCKET"`
	// RedisURL           string `env:"REDIS_URL"`
}

// Load returns a fully populated AppConfig.
// Call this once at the very beginning of main().
func Load() *AppConfig {
	// Load .env file (only in development — no error if missing)
	if err := godotenv.Load(); err != nil {
		log.Println("⚠️  No .env file found or couldn't load it. Using system environment variables only.")
	}

	cfg := &AppConfig{
		Port:               getEnv("PORT", "8080"),
		DatabaseURL:        getEnv("DATABASE_URL", "postgres://postgres:password@localhost:5432/yourdb?sslmode=disable"),
		GoogleClientID:     getEnv("GOOGLE_CLIENT_ID", ""),
		GoogleClientSecret: getEnv("GOOGLE_CLIENT_SECRET", ""),
		GoogleCallbackURL:  getEnv("GOOGLE_CALLBACK_URL", "http://localhost:8080/auth/google/callback"),
		AIProvider:         getEnv("AI_PROVIDER", "openai"),
		AIModel:            getEnv("AI_MODEL", "gpt-4o-mini"),
		AIAPIKey:           getEnv("AI_API_KEY", getEnv("OPENAI_API_KEY", "")),
		AIBaseURL:          getEnv("AI_BASE_URL", ""),
		// Add defaults for new fields here
	}

	// Required field validation (panic early with clear message)
	cfg.mustBeSet("GOOGLE_CLIENT_ID")
	cfg.mustBeSet("GOOGLE_CLIENT_SECRET")
	// Add more required fields as needed:
	// cfg.mustBeSet("DATABASE_URL")

	return cfg
}

// getEnv is the core helper from atstex-lab — clean and idiomatic.
func getEnv(key, fallback string) string {
	if val, ok := os.LookupEnv(key); ok && val != "" {
		return val
	}
	return fallback
}

// mustBeSet panics with a helpful message if a required env var is missing.
// Call this after populating the struct.
func (c *AppConfig) mustBeSet(key string) {
	switch key {
	case "GOOGLE_CLIENT_ID":
		if c.GoogleClientID == "" {
			panic("❌ Required env var GOOGLE_CLIENT_ID is not set")
		}
	case "GOOGLE_CLIENT_SECRET":
		if c.GoogleClientSecret == "" {
			panic("❌ Required env var GOOGLE_CLIENT_SECRET is not set")
		}
	// Add more cases for new required fields
	default:
		panic("unknown required key: " + key)
	}
}

// SafeLog returns a sanitized version of the config for logging (never log secrets).
func (c *AppConfig) SafeLog() map[string]any {
	return map[string]any{
		"port":                c.Port,
		"database_url":        "[REDACTED]", // or show host only
		"google_client_id":    mask(c.GoogleClientID),
		"ai_provider":         c.AIProvider,
		"ai_model":            c.AIModel,
		"ai_base_url":         c.AIBaseURL,
		// add other non-sensitive fields
	}
}

func mask(s string) string {
	if len(s) <= 8 {
		return "[REDACTED]"
	}
	return s[:4] + "..." + s[len(s)-4:]
}
```

## Usage in `main.go` (with graceful shutdown skill)

```go
func main() {
    cfg := config.Load()

    logger := logging.NewLogger(logging.Config{
        Level:       os.Getenv("LOG_LEVEL"),
        Env:         os.Getenv("APP_ENV"),
        ServiceName: "your-service",
    })
    slog.SetDefault(logger)

    logger.Info("config loaded", slog.Any("config", cfg.SafeLog()))

    if err := run(context.Background(), cfg, logger); err != nil {
        logger.Error("fatal error", slog.Any("error", err))
        os.Exit(1)
    }
}

func run(ctx context.Context, cfg *config.AppConfig, logger *slog.Logger) error {
    // Pass cfg to DB, storage, email, router, etc.
    database, err := db.New(db.Config{DSN: cfg.DatabaseURL})
    // ...
}
```

## Best Practices & Decision Tree (2026 SemmiDev standard)

**Always**:
- Call `config.Load()` **first thing** in `main()`.
- Use `cfg.SafeLog()` when logging the config (never leak secrets).
- Add new fields to `AppConfig` + `getEnv` + `SafeLog` + `mustBeSet` (if required).
- Keep defaults realistic for local development.
- Use `mustBeSet` for critical fields (fails fast with clear message).

**Decision Tree**:
1. New config value? → Add field to `AppConfig` + default in `Load()`.
2. Required? → Call `mustBeSet` after loading.
3. Secret? → Add to `SafeLog` with redaction.
4. Complex config (YAML + env)? → Only if absolutely needed (stick to env for 99% of cases).

**Never**:
- Scatter `os.Getenv` throughout the codebase.
- Load `.env` in production (it’s ignored safely).
- Log raw config (use `SafeLog`).
- Make `Load()` depend on other packages.

## Integration Notes for Your Backend Projects
- Place in `internal/config/config.go` (exact atstex-lab location).
- Combine with `graceful-shutdown-with-signal-handling` → pass `*config.AppConfig` to `run()`.
- Combine with `structured-logging-with-slog` → log sanitized config at startup.
- Combine with `database-access-with-pgx-and-sqlx` → `cfg.DatabaseURL`.
- Combine with `s3-storage-abstraction`, `smtp-email-sender-abstraction`, etc.
- Docker: mount `.env` only in dev compose file; production uses real env vars.
- Testing: override fields or use `t.Setenv` before calling `Load()`.

Follow this skill and your configuration will be:
- Extremely clean and readable
- Production-safe (no secrets in logs)
- Local-dev friendly (`.env`)
- Consistent with every SemmiDev backend (including atstex-lab)

When in doubt, copy the `AppConfig` + `Load()` + `getEnv` pattern above — this is the exact idiomatic approach used across all SemmiDev projects.

You now have a complete, professional configuration foundation! Ready for the next skill. 🚀
