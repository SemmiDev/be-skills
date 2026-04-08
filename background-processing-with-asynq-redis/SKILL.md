**Skill Name:** `background-processing-with-asynq-redis`

**Description:**  
Production-grade background job processing for Go backends using **Asynq** (Redis-backed task queue). This is the **exact pattern** used in techschool/simplebank (the reference repo) and the current 2026 Go community standard for reliable, scalable, observable async tasks.  

Asynq provides:
- Distributed workers with concurrency control
- Priority queues, retries with backoff, timeouts/deadlines
- Scheduled/delayed tasks
- Atomic enqueue-inside-transaction support
- Built-in monitoring (Inspector + optional web UI)
- Graceful shutdown
- Zero extra brokers (pure Redis)

It integrates seamlessly with your existing Clean Architecture (`domain/`, `application/`, `infrastructure/`), `config` loading, `slog` logging, `graceful-shutdown`, and `rich-domain-model`.

---

### When to use this skill
- Any long-running or non-critical task (send email, generate PDF/report, process image, sync external API, cleanup jobs, etc.).
- You need reliable delivery with retries and observability.
- You want to enqueue tasks atomically inside a database transaction (common pattern in simplebank).
- You are replacing manual goroutines, cron jobs, or custom Redis lists.

**Never** use raw `go func()` for background work in production or rely on in-memory queues.

### Installation

```bash
go get github.com/hibiken/asynq
```

(Asynq is the only dependency — Redis is already in your stack.)

### Project Structure (fits your Clean Architecture)

```bash
your-project/
├── cmd/
│   ├── server/          # existing HTTP API
│   └── worker/          # ← NEW: dedicated worker binary
│       └── main.go
├── internal/
│   ├── application/
│   │   └── worker/      # Use-case level task handlers
│   │       └── tasks.go # Business logic for each task type
│   ├── infrastructure/
│   │   └── asynq/       # Asynq client + server setup
│   │       ├── client.go
│   │       └── server.go
│   ├── domain/          # Domain events can trigger tasks
│   └── common/          # Existing random, errors, etc.
├── worker/
│   └── tasks/           # Task type constants + payload structs (optional)
│       └── types.go
├── .env.example         # Add REDIS_ADDR, etc.
└── Makefile             # Add targets: worker, worker-run
```

### Core Implementation

#### 1. Task Types & Payloads (`worker/tasks/types.go`)

```go
package tasks

const (
    TypeSendWelcomeEmail    = "email:welcome"
    TypeGenerateReport      = "report:generate"
    TypeProcessImage        = "image:process"
    // ...
)

type WelcomeEmailPayload struct {
    UserID uuid.UUID `json:"user_id"`
    Email  string    `json:"email"`
}

type GenerateReportPayload struct {
    ReportID uuid.UUID `json:"report_id"`
    UserID   uuid.UUID `json:"user_id"`
}
```

#### 2. Asynq Client (for enqueuing) — `infrastructure/asynq/client.go`

```go
package asynq

import (
    "context"
    "encoding/json"
    "time"

    "github.com/hibiken/asynq"
    "yourproject/internal/config"
)

type Client struct {
    client *asynq.Client
}

func NewClient(cfg *config.AppConfig) (*Client, error) {
    c := asynq.NewClient(asynq.RedisClientOpt{
        Addr: cfg.RedisAddr, // from your config skill
    })
    return &Client{client: c}, nil
}

func (c *Client) Enqueue(ctx context.Context, taskType string, payload any, opts ...asynq.Option) error {
    data, err := json.Marshal(payload)
    if err != nil {
        return err
    }

    task := asynq.NewTask(taskType, data, opts...)
    _, err = c.client.EnqueueContext(ctx, task)
    return err
}

// Convenience inside transaction example
func (c *Client) EnqueueAfterTx(txCtx context.Context, taskType string, payload any) error {
    return c.Enqueue(txCtx, taskType, payload, asynq.Queue("default"))
}
```

#### 3. Asynq Server & Handlers — `infrastructure/asynq/server.go`

```go
package asynq

import (
    "context"
    "log/slog"

    "github.com/hibiken/asynq"
    "yourproject/internal/application/worker"
)

type Server struct {
    srv *asynq.Server
    mux *asynq.ServeMux
}

func NewServer(cfg *config.AppConfig, appWorker *worker.Service) (*Server, error) {
    srv := asynq.NewServer(
        asynq.RedisClientOpt{Addr: cfg.RedisAddr},
        asynq.Config{
            Concurrency: 10, // adjust per CPU
            Queues: map[string]int{
                "critical": 6,
                "default":  3,
                "low":      1,
            },
            ErrorHandler: asynq.ErrorHandlerFunc(func(ctx context.Context, task *asynq.Task, err error) {
                slog.Error("task failed", slog.String("type", task.Type()), slog.Any("error", err))
            }),
            Logger: &asynqLogger{}, // wrap your slog
        },
    )

    mux := asynq.NewServeMux()
    // Register handlers from application layer
    mux.HandleFunc(tasks.TypeSendWelcomeEmail, appWorker.HandleWelcomeEmail)
    mux.HandleFunc(tasks.TypeGenerateReport, appWorker.HandleGenerateReport)
    // ...

    return &Server{srv: srv, mux: mux}, nil
}

func (s *Server) Run() error {
    return s.srv.Run(s.mux)
}

func (s *Server) Shutdown() {
    s.srv.Shutdown()
}
```

#### 4. Application Worker Service (`application/worker/tasks.go`)

```go
package worker

import (
    "context"
    "encoding/json"
    "yourproject/internal/domain"
    "yourproject/internal/infrastructure/email"
    "github.com/hibiken/asynq"
)

type Service struct {
    emailSender email.Sender
    // other dependencies
}

func (s *Service) HandleWelcomeEmail(ctx context.Context, t *asynq.Task) error {
    var p tasks.WelcomeEmailPayload
    if err := json.Unmarshal(t.Payload(), &p); err != nil {
        return err
    }

    user, err := s.userRepo.Get(ctx, p.UserID) // rich domain
    if err != nil {
        return err
    }

    return s.emailSender.Send(ctx, email.Message{
        To:      []string{user.Email},
        Subject: "Welcome!",
        // ...
    })
}
```

#### 5. Worker Entry Point (`cmd/worker/main.go`)

```go
func main() {
    cfg := config.Load()
    logger := logging.NewLogger(...) 
    slog.SetDefault(logger)

    client, _ := asynq.NewClient(cfg)           // optional, if needed
    appWorker := worker.NewService(...)         // your application worker
    srv, _ := asynq.NewServer(cfg, appWorker)

    // Graceful shutdown integration
    go func() {
        if err := srv.Run(); err != nil {
            logger.Error("worker failed", slog.Any("error", err))
        }
    }()

    // Use same graceful shutdown pattern as main server
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    srv.Shutdown()
}
```

### Enqueuing from API (inside transaction – simplebank pattern)

In your handler or application service:

```go
err := db.RunInTx(ctx, func(ctx context.Context) error {
    if err := repo.CreateUser(ctx, user); err != nil {
        return err
    }
    // Enqueue inside transaction → atomic
    return asynqClient.Enqueue(ctx, tasks.TypeSendWelcomeEmail, tasks.WelcomeEmailPayload{UserID: user.ID})
})
```

### Best Practices & Decision Tree (2026 consensus + simplebank style)

**Always**:
- Run a **separate `cmd/worker`** binary (not inside HTTP server).
- Enqueue inside DB transactions for atomicity.
- Use priority queues (`critical`/`default`/`low`).
- Make task handlers **idempotent**.
- Use `asynq.Timeout()`, `asynq.RetryDelay()`, `asynq.MaxRetry()`.
- Monitor with Asynq Inspector or the official web UI.

**Decision Tree**:
1. Simple background jobs? → Asynq (recommended, used by simplebank).
2. Need complex workflows (chains, chords)? → Consider Machinery later.
3. Ultra-minimal? → Custom Redis lists (not recommended).

**Never**:
- Run workers in the same process as HTTP server (resource contention).
- Use `math/rand` or insecure tokens in tasks.
- Forget retries and dead-letter handling.

### Integration Notes
- Add `REDIS_ADDR` to your `config.AppConfig`.
- Update `Makefile` and `Dockerfile` to build both `server` and `worker` binaries.
- Use the same graceful shutdown pattern for the worker.
- Combine with `rich-domain-model` (tasks operate on domain objects).
- Combine with `secure-random-generator` (generate OTPs/tokens inside tasks).

Follow this skill and your background processing will be:
- Reliable & observable
- Exactly like simplebank (Asynq + Redis)
- Fully integrated with your Clean Architecture

When in doubt, copy the `asynq/client.go` + `cmd/worker/main.go` + `application/worker` pattern above — this is the production standard used in simplebank and modern Go backends in 2026.
