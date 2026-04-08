---
name: feature-flags-with-flipt
description: Clean and type-safe feature flag management using Flipt (https://github.com/flipt-io/flipt). Provides a simple wrapper with boolean, string, and variant flags, caching, and context-aware evaluation (user_id, environment, etc.). Integrates with clean-idiomatic-config-loading and observability-with-opentelemetry. Always use feature flags instead of environment-based if/else for rollout, A/B testing, or kill switches.
---

# Feature Flags with Flipt

**Skill purpose**  
This skill defines the **official SemmiDev pattern** for feature flag management using **Flipt** — a self-hosted, open-source feature flag solution that is lightweight and GitOps-friendly.

### When to use this skill
- Gradual rollouts, kill switches, A/B testing, beta features.
- Environment-specific behavior without code changes.
- You want a centralized place to manage flags instead of config/env vars.

### Installation

```bash
go get github.com/flipt-io/flipt/sdk/go
```

### Setup (`internal/featureflag/client.go`)

```go
package featureflag

import (
    "context"
    "yourproject/internal/config"
    flipt "github.com/flipt-io/flipt/sdk/go"
)

type Client struct {
    client *flipt.Client
}

func NewClient(cfg *config.AppConfig) (*Client, error) {
    c, err := flipt.NewClient(flipt.WithAddress(cfg.FliptAddress))
    if err != nil {
        return nil, err
    }
    return &Client{client: c}, nil
}

// Boolean flag
func (c *Client) Bool(ctx context.Context, flagKey string, defaultValue bool, opts ...flipt.EvaluationOption) bool {
    resp, err := c.client.Boolean(ctx, flagKey, "", opts...)
    if err != nil {
        return defaultValue
    }
    return resp.Enabled
}

// String / Variant flag
func (c *Client) Variant(ctx context.Context, flagKey string, defaultValue string, opts ...flipt.EvaluationOption) string {
    resp, err := c.client.Variant(ctx, flagKey, "", opts...)
    if err != nil {
        return defaultValue
    }
    return resp.Variant
}
```

### Usage Example

```go
if ff.Bool(ctx, "new-ui-enabled", false, flipt.WithContext(map[string]string{
    "user_id": userID,
    "env":     cfg.AppEnv,
})) {
    // show new UI
}
```

### Best Practices
- Define all flags in Flipt UI or via YAML + GitOps.
- Always provide sensible default values.
- Use evaluation context (user_id, tenant, etc.) for targeted rollouts.
- Monitor flag usage with OpenTelemetry.

### Integration Notes
- Add `FliptAddress` to your `AppConfig`.
- Create the client in `NewServer` and inject into handlers/services.
- Combine with observability to track flag evaluations.
