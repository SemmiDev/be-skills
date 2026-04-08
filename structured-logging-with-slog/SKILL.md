---
name: structured-logging-with-slog
description: Production-grade, structured logging for Go backend projects using the standard library `log/slog` (Go 1.21+). Provides JSON output by default, request-correlation middleware, context propagation (request_id, trace_id), SafeError integration, sensitive-data redaction via LogValuer, and sloglint enforcement. Follows 2026 best practices from Dash0, Better Stack, and "Logging Sucks" principles: wide events, high-cardinality context, zero sensitive data leaks, and minimal log volume. Always use this instead of fmt.Printf, log.Printf, or third-party loggers like zap/zerolog unless you have extreme performance needs.
---

# Structured Logging with slog

**Skill purpose**  
This skill defines the **official SemmiDev pattern** for logging in all Go backends.  

`slog` is now the standard library choice for structured, leveled, high-performance logging. This skill gives you:
- One rich, queryable JSON event per major operation (following “wide events” philosophy).
- Automatic request correlation (`request_id`, `trace_id`, `user_id`, etc.) via context.
- Zero sensitive data leakage (passwords, tokens, full SQL, etc.).
- Seamless integration with the `secure-idiomatic-error-handling` skill (log full internal error + safe user message).
- Environment-driven config (level, format, source info).
- Testable middleware and helpers.
- Linter enforcement (`sloglint`) so every developer follows the same rules.

This replaces all legacy logging and makes your logs perfect for observability platforms (Grafana Loki, Datadog, New Relic, etc.).

## When to use this skill
- Every backend service (API, worker, CLI).
- HTTP handlers, middleware, services, repositories.
- Error paths (always log via `slog.Error` + SafeError).
- Any place where you previously used `fmt.Println`, `log.Printf`, or custom loggers.
- When you need request tracing/correlation across microservices.

**Never**:
- Use unstructured `log` package or `fmt`.
- Log raw structs without `LogValuer`.
- Log at `Debug` in production without sampling.
- Scatter 10+ log lines per request (prefer one rich event).

## Project Structure (recommended)

```
internal/common/logging/
├── logging.go      ← NewLogger, config, helpers
├── middleware.go   ← HTTP request logger + context enrichment
├── context.go      ← request_id / trace_id helpers
└── redaction.go    ← optional LogValuer helpers for common types
```

## Installation

No extra dependencies for core (pure `log/slog`).  
Recommended (add to `go.mod`):
```bash
go get github.com/veqryn/slog-context  # for easy context attributes (optional but highly recommended)
go install github.com/go-slog/sloglint/cmd/sloglint@latest  # for linting
```

Add to `.golangci.yml` (enforced best practice):
```yaml
linters:
  enable:
    - sloglint
linters-settings:
  sloglint:
    attr-only: true          # force slog.String/Int/etc.
    key-naming-case: snake   # consistent keys
    no-raw-keys: true
```

## Core Concepts

### 1. Logger Setup (`logging.go`)

```go
package logging

import (
    "context"
    "log/slog"
    "os"
    "runtime/debug"
    "time"

    "github.com/veqryn/slog-context" // optional but recommended
)

type Config struct {
    Level      string // "debug", "info", "warn", "error"
    Env        string // "development", "production", "test"
    ServiceName string
}

func NewLogger(cfg Config) *slog.Logger {
    level := parseLevel(cfg.Level)

    opts := &slog.HandlerOptions{
        Level:     level,
        AddSource: cfg.Env == "development", // stack in dev only
        ReplaceAttr: func(groups []string, a slog.Attr) slog.Attr {
            // Custom level names, redaction, etc.
            if a.Key == slog.LevelKey {
                // optional: map to TRACE / FATAL if you added custom levels
            }
            // Redact known sensitive keys (see redaction.go)
            if isSensitiveKey(a.Key) {
                return slog.String(a.Key, "[REDACTED]")
            }
            return a
        },
    }

    var handler slog.Handler
    if cfg.Env == "production" || cfg.Env == "staging" {
        handler = slog.NewJSONHandler(os.Stdout, opts)
    } else {
        handler = slog.NewTextHandler(os.Stdout, opts)
    }

    // Wrap with context handler for automatic request_id / trace_id
    handler = slogcontext.NewHandler(handler)

    logger := slog.New(handler)

    // Add global service metadata once
    logger = logger.With(
        slog.String("service", cfg.ServiceName),
        slog.String("env", cfg.Env),
        slog.String("version", getBuildVersion()),
    )

    slog.SetDefault(logger) // optional: makes slog.Info() work everywhere
    return logger
}

func parseLevel(l string) slog.Level {
    switch l {
    case "debug":
        return slog.LevelDebug
    case "warn":
        return slog.LevelWarn
    case "error":
        return slog.LevelError
    default:
        return slog.LevelInfo
    }
}

func getBuildVersion() string {
    if info, ok := debug.ReadBuildInfo(); ok {
        return info.Main.Version
    }
    return "dev"
}
```

### 2. Context Helpers (`context.go`)

```go
package logging

import (
    "context"

    "github.com/google/uuid"
    "github.com/veqryn/slog-context"
)

type contextKey string

const requestIDKey contextKey = "request_id"

func WithRequestID(ctx context.Context) context.Context {
    id := uuid.New().String()
    return slogctx.Prepend(ctx, slog.String("request_id", id))
}

func GetRequestID(ctx context.Context) string {
    if id, ok := ctx.Value(requestIDKey).(string); ok {
        return id
    }
    return "unknown"
}

// For distributed tracing (OpenTelemetry compatible)
func WithTraceID(ctx context.Context, traceID string) context.Context {
    return slogctx.Prepend(ctx, slog.String("trace_id", traceID))
}
```

### 3. HTTP Middleware (`middleware.go`) – Wide Event Pattern

```go
package logging

import (
    "log/slog"
    "net/http"
    "time"
)

func LoggingMiddleware(logger *slog.Logger) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            start := time.Now()

            // Enrich context once
            ctx := r.Context()
            ctx = WithRequestID(ctx) // or extract from header
            r = r.WithContext(ctx)

            // Optional: wrap response writer to capture status
            rw := &responseWriter{ResponseWriter: w, status: http.StatusOK}

            logger.InfoContext(ctx, "request started",
                slog.String("method", r.Method),
                slog.String("path", r.URL.Path),
                slog.String("remote_ip", r.RemoteAddr),
                slog.String("user_agent", r.UserAgent()),
            )

            next.ServeHTTP(rw, r)

            duration := time.Since(start).Milliseconds()

            level := slog.LevelInfo
            if rw.status >= 500 {
                level = slog.LevelError
            } else if rw.status >= 400 {
                level = slog.LevelWarn
            }

            logger.LogAttrs(ctx, level, "request completed",
                slog.Int("status", rw.status),
                slog.Int64("duration_ms", duration),
                slog.String("method", r.Method),
                slog.String("path", r.URL.Path),
            )
        })
    }
}

type responseWriter struct {
    http.ResponseWriter
    status int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.status = code
    rw.ResponseWriter.WriteHeader(code)
}
```

## Recommended Usage Patterns

### 1. Setup (in `main.go`)
```go
func main() {
    logger := logging.NewLogger(logging.Config{
        Level:       os.Getenv("LOG_LEVEL"),
        Env:         os.Getenv("APP_ENV"),
        ServiceName: "my-service",
    })

    mux := http.NewServeMux()
    mux.Handle("/", logging.LoggingMiddleware(logger)(yourHandler))

    // ...
}
```

### 2. In Services / Handlers (with SafeError integration)
```go
func GetUser(ctx context.Context, id string) (*User, error) {
    user, err := repo.Get(ctx, id)
    if err != nil {
        var safe *errors.SafeError
        if errors.As(err, &safe) {
            // Log full internal details + safe message
            slog.ErrorContext(ctx, "failed to get user",
                slog.String("code", safe.Code),
                slog.String("user_msg", safe.UserMsg),
                slog.Any("error", safe), // uses Unwrap + LogValuer if implemented
                slog.String("user_id", id),
            )
        }
        return nil, err
    }
    return user, nil
}
```

### 3. Custom Types (redaction)
```go
// In redaction.go or on your DTOs
func (u *User) LogValue() slog.Value {
    return slog.GroupValue(
        slog.String("id", u.ID),
        slog.String("email", "[REDACTED]"), // or omit
    )
}
```

## Best Practices & Decision Tree (2026 consensus)

**Always**:
- Use `slog.String`, `slog.Int`, etc. (enforced by linter).
- Add high-cardinality fields (`request_id`, `user_id`, `trace_id`).
- Log one rich event per request/operation (wide event).
- Use context-aware methods (`InfoContext`, `ErrorContext`).
- Check `logger.Enabled(ctx, level)` for expensive debug logs.
- Redact sensitive data via `LogValuer` or `ReplaceAttr`.

**Decision Tree**:
1. New request? → LoggingMiddleware handles correlation.
2. Business logic? → Log at appropriate level with full context.
3. Error occurred? → Always use SafeError + slog.ErrorContext.
4. High traffic? → Consider sampling (add `slog-sampling` middleware later).
5. Dev vs Prod? → Text in dev (readable), JSON in prod (machine-friendly).

**Never** (from "Logging Sucks"):
- Log fragmented low-context lines.
- Log raw passwords, tokens, full SQL queries.
- Use string formatting (`%s`) for structured data.
- Log everything at Debug in production.

## Integration Notes for Your Backend Projects
- Call `NewLogger` once in `main`.
- Add `LoggingMiddleware` to every router.
- Combine with `secure-idiomatic-error-handling` → perfect error logs.
- Combine with `http-error-handling-with-problem` → log before writing Problem response.
- For distributed tracing → extend `WithTraceID` with OpenTelemetry.
- Monitor log volume and add sampling only if needed.
- Document your common log keys in README (request_id, user_id, etc.).

Follow this skill and every log in your Go backends will be:
- Structured & queryable
- Secure (no leaks)
- Correlated across services
- Minimal & high-signal
- Production-ready out of the box

When in doubt, copy the `NewLogger` + `LoggingMiddleware` + `slog.ErrorContext` pattern above — this is the exact design now used across all SemmiDev backend projects.
