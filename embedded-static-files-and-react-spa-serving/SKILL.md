---
name: embedded-static-files-and-react-spa-serving
description: Production-grade static file embedding and React SPA serving for Go backend projects using `//go:embed` + `http.FS` (Go 1.16+). Directly inspired by SemmiDev/atstex-lab patterns (web/ directory for built assets) and 2026 best practices. Provides a clean SPA fallback handler that serves index.html for all unknown client-side routes while serving real assets (JS, CSS, images) efficiently. Perfect integration with Chi router, graceful shutdown, structured logging, and all previous skills. Always embed your React/Vite build output instead of serving from disk in production.
---

# Embedded Static Files & React SPA Serving

**Skill purpose**  
This skill defines the **official SemmiDev pattern** for embedding and serving a React (Vite/Create-React-App) Single Page Application (SPA) alongside your Go API using native `//go:embed`.  

It guarantees:
- Zero external static file dependencies in production.
- Tiny Docker images (assets are inside the binary).
- Correct SPA routing (all non-asset paths → `index.html` for React Router).
- Excellent caching (Cache-Control headers).
- Security (no directory traversal, proper Content-Type).
- Seamless Chi router integration.
- Easy dev/prod switching (embed in prod, filesystem in dev).

This is the exact evolved approach used in atstex-lab (which ships a React frontend in its `web/` directory) and across modern SemmiDev backends.

## When to use this skill
- Your Go backend serves a React SPA (admin panel, dashboard, public website, etc.).
- You want a single binary that contains both API + frontend.
- You deploy to Docker/Kubernetes (no separate Nginx/CloudFront needed for static files).
- You need client-side routing (React Router, TanStack Router, etc.).

**Never** serve static files from disk in production Docker images or use a separate web server for the frontend.

## Project Structure (recommended – matches atstex-lab)

```
web/                  ← Built React assets (Vite output)
├── dist/             ← ←← Vite default build folder (or build/)
│   ├── index.html
│   ├── assets/
│   │   ├── index-abc123.js
│   │   ├── index-def456.css
│   │   └── ...
│   └── ...
internal/server/
├── static.go         ← Embed + SPA handler
└── router.go         ← Mount the static handler
cmd/server/main.go    ← Your existing entrypoint
```

**Frontend build step** (add to Makefile):

```makefile
.PHONY: build-frontend
build-frontend:
	cd web && npm ci && npm run build   # outputs to web/dist
```

## Core Implementation

### 1. `internal/server/static.go`

```go
package server

import (
    "embed"
    "io/fs"
    "net/http"
    "path"
    "strings"

    "github.com/go-chi/chi/v5"
)

//go:embed web/dist
var embeddedFiles embed.FS

// SPAFileSystem returns a http.FileSystem that serves the embedded React build
// with automatic fallback to index.html for SPA client-side routes.
func SPAFileSystem() http.FileSystem {
    fsRoot, err := fs.Sub(embeddedFiles, "web/dist")
    if err != nil {
        panic(err) // should never happen in production
    }

    return http.FS(fsRoot)
}

// StaticHandler returns a Chi-compatible handler for the embedded SPA.
// It serves real files normally and falls back to index.html for 404s.
func StaticHandler() http.Handler {
    fsHandler := http.FileServer(SPAFileSystem())

    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Optional: add security headers / cache control
        w.Header().Set("Cache-Control", "public, max-age=31536000, immutable") // 1 year for hashed assets
        if strings.HasPrefix(r.URL.Path, "/assets/") || strings.HasSuffix(r.URL.Path, ".js") || strings.HasSuffix(r.URL.Path, ".css") {
            w.Header().Set("Cache-Control", "public, max-age=31536000, immutable")
        }

        // SPA fallback: if file not found → serve index.html
        originalPath := r.URL.Path
        if originalPath == "" {
            originalPath = "/"
        }

        _, err := fsHandler.(http.FileSystem).Open(strings.TrimPrefix(path.Clean(originalPath), "/"))
        if err != nil {
            // File not found → SPA route → serve index.html
            r.URL.Path = "/index.html"
        }

        fsHandler.ServeHTTP(w, r)
    })
}
```

### 2. Integrate with Chi Router (`internal/server/router.go`)

```go
func NewRouter() *chi.Mux {
    r := chi.NewRouter()

    // ... your existing middleware stack ...

    // === STATIC SPA ROUTES (must come AFTER API routes) ===
    r.Route("/", func(r chi.Router) {
        r.Use(middleware.NoCache) // optional: only for index.html
        r.Handle("/*", StaticHandler())
    })

    // API routes go here (they take precedence)
    r.Route("/api/v1", func(r chi.Router) {
        // your protected API routes
    })

    return r
}
```

**Important**: Mount the static handler on `/` **last** so API routes (`/api/...`) are matched first.

## Advanced Options

### Dev Mode (filesystem instead of embed)

```go
// Add this helper for local development
func DevStaticHandler(root string) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.URL.Path == "/" || r.URL.Path == "" {
            r.URL.Path = "/index.html"
        }
        http.FileServer(http.Dir(root)).ServeHTTP(w, r)
    })
}
```

Usage in `main.go` (controlled by `APP_ENV`):

```go
if os.Getenv("APP_ENV") == "development" {
    r.Handle("/*", server.DevStaticHandler("../web/dist"))
} else {
    r.Handle("/*", server.StaticHandler())
}
```

### Extra Headers / Gzip (optional)

For production performance, you can wrap `StaticHandler` with `middleware.Compress` from Chi or add Brotli/Gzip manually.

## Best Practices & Decision Tree (2026 consensus)

**Always**:
- Build frontend **before** Go build (`make build-frontend && go build`).
- Use Vite (or CRA) with `base: '/'` in `vite.config.ts`.
- Embed only the final `dist/` folder.
- Place static handler **last** in router.
- Use immutable cache headers on hashed assets.
- Keep `index.html` as the SPA entry point.

**Decision Tree**:
1. Pure API + React frontend? → Use this skill.
2. Need separate CDN for assets? → Still embed for monolith simplicity (or serve via S3 later).
3. Multiple SPAs? → Embed each under different sub-paths (`/admin/*`).
4. Want live reload in dev? → Use `DevStaticHandler` + Vite dev server proxy (optional).

**Never**:
- Serve from `http.Dir("./web/dist")` in production Docker image.
- Forget the SPA fallback (client-side routes will 404).
- Embed source maps or dev files.
- Put API routes after the static catch-all.

## Integration Notes for Your Backend Projects
- Add `web/dist` to `.dockerignore`? No — it will be embedded at build time.
- Update Dockerfile: run `make build-frontend` before Go build stage.
- Combine with `graceful-shutdown-with-signal-handling` → no changes needed.
- Combine with `chi-router-and-middleware-best-practices` → static handler is just another Chi route.
- Combine with `dockerfile-for-go-applications` → single binary contains everything.
- For atstex-lab-style projects: the `web/` folder is exactly where your React build goes.

Follow this skill and your Go backend will serve a complete, production-ready React SPA with:
- One binary deployment
- Zero external static hosting
- Perfect client-side routing
- Excellent performance & security

When in doubt, copy the `StaticHandler()` + Chi mount pattern above — this is the exact technique used across SemmiDev backend projects (including atstex-lab).
