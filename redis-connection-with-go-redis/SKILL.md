---
name: redis-connection-with-go-redis
description: Production-grade, idiomatic Redis connection management for Go backend projects using the official `github.com/redis/go-redis/v9` client. Provides a clean singleton client with proper connection pooling, timeouts, TLS support, health checking, and graceful shutdown. Follows 2026 best practices (pool tuning based on GOMAXPROCS, context-aware operations, error wrapping with SafeError). Integrates seamlessly with clean-idiomatic-config-loading, graceful-shutdown-with-signal-handling, background-processing-with-asynq-redis, and your Clean Architecture. Always use a single shared client instance created at startup instead of creating new clients per request.
---

# Redis Connection with go-redis (Best Practices)

**Skill purpose**  
This skill defines the **official SemmiDev pattern** for connecting to Redis using `github.com/redis/go-redis/v9` (the most widely used and officially recommended client in 2026).  

It emphasizes:
- A **single shared client** with built-in connection pooling (no manual pools needed).
- Sensible defaults + configurable tuning via `config.AppConfig`.
- Context-aware operations everywhere.
- Health check on startup + graceful close on shutdown.
- Secure defaults (TLS support, timeouts, no blocking forever).
- Clean integration with your existing config, logging, and error handling.

**Note on alternatives**: While `rueidis` offers higher performance with auto-pipelining and client-side caching, `go-redis/v9` remains the standard choice for most projects due to its stability, ecosystem, and simplicity. Use this skill unless you have extreme throughput requirements.

## When to use this skill
- Caching, rate limiting, session storage, distributed locks, queues (including Asynq).
- Any background processing or real-time features that need Redis.
- You want a production-ready Redis client with proper pooling and observability.

**Never** create a new Redis client per request or use `math/rand` for anything in Redis.

## Installation

```bash
go get github.com/redis/go-redis/v9
```

## Project Structure

```
internal/
├── config/
│   └── config.go          # Add RedisAddr, RedisPassword, RedisDB, etc.
├── infrastructure/
│   └── redis/
│       └── client.go      ← Core Redis client wrapper
└── common/                # If you want shared Redis helpers
```

## 1. Extend Config (`internal/config/config.go`)

Add these fields to your `AppConfig`:

```go
type AppConfig struct {
    // ... existing fields

    // Redis
    RedisAddr      string `env:"REDIS_ADDR"`      // e.g. "localhost:6379" or "redis:6379"
    RedisPassword  string `env:"REDIS_PASSWORD"`
    RedisDB        int    `env:"REDIS_DB"`        // default 0
    RedisPoolSize  int    `env:"REDIS_POOL_SIZE"` // 0 = auto (GOMAXPROCS * 10)
}
```

In `Load()`:

```go
RedisAddr:     getEnv("REDIS_ADDR", "localhost:6379"),
RedisPassword: getEnv("REDIS_PASSWORD", ""),
RedisDB:       getEnvInt("REDIS_DB", 0),
RedisPoolSize: getEnvInt("REDIS_POOL_SIZE", 0), // 0 means use library default
```

## 2. Redis Client Wrapper (`internal/infrastructure/redis/client.go`)

```go
package redis

import (
    "context"
    "fmt"
    "time"

    "github.com/redis/go-redis/v9"
    "yourproject/internal/config"
    "yourproject/internal/common/errors" // your secure error skill
    "log/slog"
)

type Client struct {
    *redis.Client
}

func NewClient(cfg *config.AppConfig, logger *slog.Logger) (*Client, error) {
    opt := &redis.Options{
        Addr:     cfg.RedisAddr,
        Password: cfg.RedisPassword,
        DB:       cfg.RedisDB,

        // Connection pool tuning (best practice)
        PoolSize:     cfg.RedisPoolSize,           // 0 = runtime.GOMAXPROCS() * 10 (recommended default)
        MinIdleConns: 5,                           // Keep some idle connections warm
        PoolTimeout:  30 * time.Second,            // Wait time to get connection from pool

        // Timeouts (critical for production)
        DialTimeout:  5 * time.Second,
        ReadTimeout:  3 * time.Second,
        WriteTimeout: 3 * time.Second,

        // Enable RESP3 if Redis >= 6 (better performance)
        Protocol: 3,
    }

    client := redis.NewClient(opt)

    // Health check on startup
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    if err := client.Ping(ctx).Err(); err != nil {
        return nil, fmt.Errorf("redis ping failed: %w", err)
    }

    logger.Info("connected to redis",
        slog.String("addr", cfg.RedisAddr),
        slog.Int("db", cfg.RedisDB),
    )

    return &Client{Client: client}, nil
}

// Close for graceful shutdown
func (c *Client) Close() error {
    return c.Client.Close()
}

// Helper to wrap errors with your SafeError pattern
func (c *Client) WrapError(err error, op string) error {
    if err == nil {
        return nil
    }
    return errors.New("REDIS_ERROR", fmt.Sprintf("redis %s failed", op), err, nil)
}
```

## 3. Usage in `main.go` (with graceful shutdown)

```go
func run(ctx context.Context, cfg *config.AppConfig, logger *slog.Logger) error {
    redisClient, err := redis.NewClient(cfg, logger)
    if err != nil {
        return err
    }
    defer redisClient.Close()

    // Pass to Asynq, cache layer, session store, etc.
    asynqClient := asynq.NewClient(...) // uses same Redis

    // ... rest of your setup

    // Graceful shutdown will call Close()
    return nil
}
```

## Best Practices & Decision Tree (2026 consensus)

**Always**:
- Create **one client instance** at startup and share it everywhere.
- Tune `PoolSize` based on your workload (default is usually excellent).
- Use context with timeout for every Redis operation.
- Enable `Protocol: 3` for modern Redis.
- Close the client during graceful shutdown.

**Decision Tree**:
1. Simple caching / sessions? → Use this skill with `go-redis/v9`.
2. Extreme performance + client-side caching? → Consider switching to `rueidis` later (different API).
3. Redis Cluster / Sentinel? → `go-redis` supports both natively (`NewClusterClient`, `NewFailoverClient`).

**Never**:
- Create a new client per request (destroys pooling benefits).
- Ignore timeouts (can cause hanging goroutines).
- Use blocking commands without careful consideration.

## Integration Notes for Your Backend Projects
- Add Redis fields to `config.AppConfig` and `SafeLog()`.
- Place the wrapper in `internal/infrastructure/redis/client.go`.
- Combine with `background-processing-with-asynq-redis` (Asynq uses the same Redis options).
- Combine with `graceful-shutdown-with-signal-handling` → call `redisClient.Close()`.
- For local dev: use Docker Compose with Redis.
- Monitoring: Use `redisClient.Info()` periodically or Asynq's built-in metrics.

Follow this skill and your Redis connections will be:
- Efficient (proper pooling)
- Reliable (timeouts + health check)
- Secure & observable
- Consistent with all SemmiDev backends

When in doubt, use the `NewClient` wrapper above — this is the clean, production-ready pattern aligned with go-redis official recommendations and simplebank-style architecture.
