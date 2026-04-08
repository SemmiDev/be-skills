```markdown
---
name: observability-with-opentelemetry
description: Production-grade observability for Go backend services using OpenTelemetry (OTel) with tracing, metrics, and logging correlation. Provides automatic instrumentation for HTTP (Chi), Redis, PostgreSQL (pgx), and Asynq. Includes trace ID propagation, structured logging correlation with slog, and export to Jaeger/Tempo + Prometheus. Integrates with structured-logging-with-slog, graceful-shutdown, and all previous skills. Always enable OpenTelemetry from the beginning for production services.
---

# Observability with OpenTelemetry

**Skill purpose**  
This skill defines the **official SemmiDev pattern** for full observability using **OpenTelemetry** (the industry standard in 2026). It provides distributed tracing, metrics, and log correlation across HTTP, database, Redis, background jobs (Asynq), and custom business operations.

### When to use this skill
- Any production service that needs tracing, metrics, and request correlation.
- You want to observe latency, error rates, and dependencies (DB, Redis, external APIs).
- You are using Jaeger, Tempo, Prometheus, or Grafana Stack.

### Installation

```bash
go get go.opentelemetry.io/otel
go get go.opentelemetry.io/otel/sdk
go get go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp
go get go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp
go get go.opentelemetry.io/contrib/instrumentation/github.com/go-chi/chi/otelchi
```

### Core Setup (`internal/observability/otel.go`)

```go
package observability

import (
    "context"
    "time"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp"
    "go.opentelemetry.io/otel/propagation"
    "go.opentelemetry.io/otel/sdk/resource"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.21.0"
)

type Config struct {
    OTLPEndpoint string
    ServiceName  string
    Environment  string
}

func InitOTel(cfg Config) (func(context.Context) error, error) {
    exporter, err := otlptracehttp.New(context.Background(),
        otlptracehttp.WithEndpoint(cfg.OTLPEndpoint),
        otlptracehttp.WithInsecure(),
    )
    if err != nil {
        return nil, err
    }

    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter),
        sdktrace.WithResource(resource.NewWithAttributes(
            semconv.SchemaURL,
            semconv.ServiceNameKey.String(cfg.ServiceName),
            semconv.DeploymentEnvironmentKey.String(cfg.Environment),
        )),
    )

    otel.SetTracerProvider(tp)
    otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{},
        propagation.Baggage{},
    ))

    return tp.Shutdown, nil
}
```

### Integration in `main.go`

```go
shutdown, err := observability.InitOTel(observability.Config{
    OTLPEndpoint: cfg.OTLPEndpoint,
    ServiceName:  "your-service",
    Environment:  cfg.AppEnv,
})
if err != nil { ... }
defer shutdown(context.Background())
```

### Automatic Instrumentation

- **HTTP (Chi)**: Use `otelchi.Middleware`
- **pgx**: Use `github.com/jackc/pgx/v5/tracetrace`
- **Redis (go-redis)**: Use `github.com/redis/go-redis/extra/redisotel`
- **Asynq**: Wrap handlers with tracing

### Best Practices
- Always propagate trace context.
- Add custom spans for important business operations.
- Correlate logs with trace ID using your slog middleware.
- Use semantic conventions.
