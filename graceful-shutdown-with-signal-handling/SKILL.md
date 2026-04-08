---
name: graceful-shutdown-with-signal-handling
description: Production-grade graceful shutdown for Go backend services using the exact pattern from SemmiDev/atstex-lab/cmd/server/main.go (signal.Notify + errc channel + http.Server.Shutdown with timeout). Covers full resource cleanup (DB pool, storage, email sender, etc.) in reverse initialization order, structured slog logging, and context-aware cancellation. Works perfectly with Chi router, pgxpool, and all previous skills. Always use this pattern in main.go instead of raw ListenAndServe or ignoring signals.
---

# Graceful Shutdown with Signal Handling

**Skill purpose**  
This skill defines the **official SemmiDev pattern** for graceful shutdown, directly based on the production implementation in `atstex-lab/cmd/server/main.go`.  

It ensures:
- The server stops accepting new requests immediately on SIGINT/SIGTERM (Kubernetes, Docker, systemd).
- All in-flight HTTP requests finish (or timeout).
- Resources are closed cleanly in reverse order of creation (DB → storage → email → etc.).
- Shutdown is logged with `slog` for observability.
- A configurable timeout prevents hanging (default 15–30s).
- The process exits with code 0 on clean shutdown, 1 on error.

This pattern is battle-tested, Kubernetes/Docker-native, and integrates seamlessly with your Chi router, pgx + sqlx, slog logging, SafeError handling, and all other skills.

## When to use this skill
- Every Go HTTP service (`cmd/server/main.go`, workers, CLI tools with background servers).
- You deploy to containers, Kubernetes, Cloud Run, or any orchestrator that sends SIGTERM.
- You have long-running requests, open DB connections, S3 clients, SMTP connections, etc.

**Never**:
- Call `os.Exit` directly.
- Ignore shutdown signals.
- Let the process die abruptly (kills in-flight work and can corrupt state).

## Core Pattern (exact evolution of atstex-lab)

### 1. `main.go` (canonical template)

```go
package main

import (
    "context"
    "fmt"
    "log/slog"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "yourproject/internal/common/db"
    "yourproject/internal/common/logging"
    "yourproject/internal/common/storage"   // if using S3
    "yourproject/internal/common/email"     // if using SMTP
    "yourproject/internal/server"
    // add your other packages
)

func main() {
    logger := logging.NewLogger(logging.Config{
        Level:       os.Getenv("LOG_LEVEL"),
        Env:         os.Getenv("APP_ENV"),
        ServiceName: "your-service",
    })
    slog.SetDefault(logger)

    if err := run(context.Background(), logger); err != nil {
        logger.Error("server shutdown with error", slog.Any("error", err))
        os.Exit(1)
    }
    logger.Info("server exited cleanly")
}

func run(ctx context.Context, logger *slog.Logger) error {
    // === 1. Initialize resources (in creation order) ===
    cfg := loadConfig() // your config skill or env

    database, err := db.New(db.Config{DSN: cfg.DatabaseURL})
    if err != nil {
        return fmt.Errorf("connect to db: %w", err)
    }
    defer database.Close() // safety net

    s3Storage, err := storage.NewS3Storage(storage.S3Config{...})
    if err != nil {
        return fmt.Errorf("init s3: %w", err)
    }

    emailSender, err := email.NewSMTP(email.Config{...})
    if err != nil {
        return fmt.Errorf("init email: %w", err)
    }

    // Validator, other singletons, etc.
    // ...

    // === 2. Create HTTP server ===
    router := server.NewRouter() // from chi-router skill
    httpSrv := &http.Server{
        Addr:         ":" + cfg.Port,
        Handler:      router,
        ReadTimeout:  15 * time.Second,
        WriteTimeout: 120 * time.Second, // allow long responses (exports, etc.)
        IdleTimeout:  60 * time.Second,
    }

    // === 3. Start server in background ===
    errc := make(chan error, 1)
    go func() {
        logger.Info("starting server", slog.String("addr", httpSrv.Addr))
        if err := httpSrv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            errc <- err
        }
    }()

    // === 4. Wait for shutdown signal ===
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)

    select {
    case err := <-errc:
        return fmt.Errorf("server listen error: %w", err)
    case sig := <-quit:
        logger.Info("shutdown signal received", slog.String("signal", sig.String()))
    }

    // === 5. Graceful shutdown with timeout ===
    shutdownCtx, cancel := context.WithTimeout(ctx, 30*time.Second) // adjust per your needs
    defer cancel()

    logger.Info("shutting down http server...")
    if err := httpSrv.Shutdown(shutdownCtx); err != nil {
        logger.Error("http server shutdown failed", slog.Any("error", err))
        // still continue to close other resources
    } else {
        logger.Info("http server stopped cleanly")
    }

    // === 6. Close resources in REVERSE order of creation ===
    logger.Info("closing resources...")

    if err := emailSender.Close(); err != nil { // if your email lib has Close
        logger.Error("failed to close email sender", slog.Any("error", err))
    }
    if err := s3Storage.Close(); err != nil { // if your storage has Close
        logger.Error("failed to close s3 storage", slog.Any("error", err))
    }
    database.Close() // already deferred, but explicit is fine

    logger.Info("all resources closed")
    return nil
}
```

## Best Practices & Decision Tree (2026 consensus)

**Always**:
- Use the exact `run()` + `errc` + `quit` + `Shutdown` pattern from atstex-lab.
- Set sensible `ReadTimeout`/`WriteTimeout`/`IdleTimeout` on `http.Server`.
- Log every step with `slog` (including signal name).
- Close resources in **reverse** initialization order.
- Use `defer database.Close()` as safety net.
- Return errors from `run()` so main can exit with code 1.

**Decision Tree**:
1. Simple HTTP API? → Use the template above.
2. Background workers/goroutines? → Add them to a `sync.WaitGroup` and wait in shutdown.
3. Need custom shutdown per resource? → Give each resource a `Close() error` or `Shutdown(ctx) error` method.
4. Kubernetes/Docker? → 15–30s timeout is perfect (matches default terminationGracePeriodSeconds).
5. Long-running requests (file uploads, reports)? → Increase `WriteTimeout` and shutdown timeout.

**Never**:
- Block `main()` with `ListenAndServe` directly.
- Use `httpSrv.Close()` (forceful) unless `Shutdown` fails.
- Forget to handle `http.ErrServerClosed`.
- Log at Error level for normal SIGTERM.

## Integration Notes for Your Backend Projects
- Place this in `cmd/server/main.go` (or `cmd/api/main.go`).
- Reuse the same `run()` pattern for workers/background services.
- Combine with `structured-logging-with-slog` → all shutdown steps are observable.
- Combine with `database-access-with-pgx-and-sqlx` → `database.Close()` calls `pgxpool.Close()`.
- Combine with `s3-storage-abstraction`, `smtp-email-sender-abstraction` → add their `Close()` if implemented.
- Docker/K8s: Use `CMD ["./server"]` (exec form) so signals reach the Go process.
- Testing: You can call `run()` in integration tests with a short timeout.

Follow this skill and every service in your Go backends will shut down **cleanly, safely, and observably** — exactly as done in all SemmiDev projects (including atstex-lab).

When in doubt, copy the `run()` function above — this is the canonical, production-proven pattern.
