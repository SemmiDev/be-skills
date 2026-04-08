---
name: server-side-html-rendering-with-templates
description: Production-grade server-side HTML rendering for Go backends using `html/template` with clean layouts, partials, and embedding. Directly based on the SemmiDev/atstex-lab/web structure (templates/ + static/ + embed.go). Provides a reusable TemplateRenderer with pre-parsed templates, layout inheritance via {{define}}/{{template}}, custom FuncMap, and seamless Chi router integration. Perfect for admin panels, marketing pages, or hybrid apps where React SPA is overkill. Always use this instead of raw template parsing or embedding React for simple HTML pages.
---

# Server-Side HTML Rendering with Templates

**Skill purpose**  
This skill defines the **official SemmiDev pattern** for server-side HTML pages using Go's `html/template` package.  

It follows the exact structure from `atstex-lab/web` (templates/ folder + embed.go) while incorporating 2026 best practices:
- Layout inheritance (`base.html` + `{{define "content"}}`)
- Reusable partials (header, footer, nav, etc.)
- Pre-parsed template cache for performance
- `//go:embed` for zero-disk dependency in production
- Custom `FuncMap` for helpers (format date, safe HTML, etc.)
- Clean separation: layouts/, partials/, pages/
- Full integration with Chi router, graceful shutdown, and your existing skills

This gives you fast, secure, maintainable server-rendered pages — ideal for admin dashboards, email previews, marketing landing pages, or any non-SPA UI.

## When to use this skill
- You need traditional server-rendered HTML pages (not a full React SPA).
- You have an admin panel, public marketing site, or hybrid frontend.
- You want layout reuse without duplicating head/nav/footer.
- You are replacing manual `template.ParseFiles` calls or third-party template engines.

**Never** parse templates on every request or serve raw `.html` files from disk in production.

## Project Structure (exact match to atstex-lab/web)

```
web/
├── embed.go               ← //go:embed + helper functions
├── static/                ← CSS, JS, images (embedded or served separately)
│   ├── css/
│   ├── js/
│   └── img/
└── templates/
    ├── layouts/
    │   └── base.html      ← Master layout with {{define "layout"}}
    ├── partials/
    │   ├── header.html
    │   ├── footer.html
    │   ├── nav.html
    │   └── ...
    └── pages/
        ├── home.html      ← Defines {{define "content"}}
        ├── about.html
        ├── dashboard.html
        └── ...
```

## Core Implementation

### 1. `web/embed.go`

```go
package web

import (
    "embed"
    "html/template"
    "io/fs"
    "net/http"
)

//go:embed templates static
var embeddedFiles embed.FS

// Templates returns a pre-parsed *template.Template with all layouts + pages + partials
var Templates *template.Template

// InitTemplates must be called once at startup (in main or server init)
func InitTemplates() {
    t, err := template.ParseFS(embeddedFiles,
        "templates/layouts/*.html",
        "templates/partials/*.html",
        "templates/pages/*.html",
    )
    if err != nil {
        panic("failed to parse templates: " + err.Error())
    }
    Templates = t
}

// Render is the canonical helper used by all handlers
func Render(w http.ResponseWriter, name string, data any) error {
    if Templates == nil {
        return http.ErrNotSupported // or custom error
    }
    w.Header().Set("Content-Type", "text/html; charset=utf-8")
    return Templates.ExecuteTemplate(w, name, data)
}
```

### 2. Layout Pattern (templates/layouts/base.html)

```html
{{define "layout"}}
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{{.Title}}</title>
    <!-- CSS from static/ -->
    <link rel="stylesheet" href="/static/css/main.css">
    {{block "head" .}}{{end}}
</head>
<body>
    {{template "partials/nav" .}}
    <main>
        {{template "content" .}}   <!-- page-specific content goes here -->
    </main>
    {{template "partials/footer" .}}
    <script src="/static/js/main.js"></script>
    {{block "scripts" .}}{{end}}
</body>
</html>
{{end}}
```

### 3. Page Example (templates/pages/home.html)

```html
{{define "content"}}
<h1>Welcome to Atstex Lab</h1>
<p>{{.Message}}</p>
{{end}}

{{define "head"}}
<meta name="description" content="CV builder">
{{end}}
```

### 4. Partials (templates/partials/nav.html)

```html
{{define "partials/nav"}}
<nav>...</nav>
{{end}}
```

### 5. Chi Router Integration (internal/server/router.go)

```go
func NewRouter() *chi.Mux {
    r := chi.NewRouter()

    // ... middleware stack ...

    // Static assets from embedded folder
    r.Handle("/static/*", http.StripPrefix("/static/", http.FileServer(http.FS(web.StaticFS()))))

    // Server-rendered pages (use Render helper)
    r.Get("/", func(w http.ResponseWriter, r *http.Request) {
        data := map[string]any{
            "Title":   "Home",
            "Message": "Hello from server-side rendered page",
        }
        if err := web.Render(w, "layout", data); err != nil {  // "layout" is the base
            http.Error(w, err.Error(), http.StatusInternalServerError)
        }
    })

    // API routes come after (or in /api group)
    r.Route("/api/v1", func(r chi.Router) { ... })

    return r
}
```

**Helper for static FS** (add to web/embed.go):

```go
func StaticFS() http.FileSystem {
    sub, _ := fs.Sub(embeddedFiles, "static")
    return http.FS(sub)
}
```

## Best Practices & Decision Tree (2026 consensus)

**Always**:
- Call `web.InitTemplates()` once in `main()` or server setup (before router creation).
- Use `{{define "layout"}}` + `{{template "content" .}}` for inheritance.
- Put reusable pieces in `partials/`.
- Use `html/template` (never `text/template`) for security (auto-escaping).
- Pre-parse with `ParseFS` + `embed.FS`.
- Pass data as `map[string]any` or dedicated structs.
- Add custom `FuncMap` for helpers (e.g. `formatDate`, `safeHTML`).

**Decision Tree**:
1. Simple page? → `web.Render(w, "layout", data)`
2. Need extra scripts/CSS per page? → Use `{{block "head"}}` / `{{block "scripts"}}`
3. Want component-like reuse? → Partials + `{{template "partials/xxx" .}}`
4. Hybrid (some React)? → Keep React in `web/dist/` (previous skill) and server-rendered pages in `templates/`
5. Production? → Always embed; never `http.Dir`

**Never**:
- Parse templates inside handlers (slow + error-prone).
- Use string concatenation for HTML.
- Forget to call `InitTemplates()` before starting the server.
- Serve templates from disk in Docker (use embed).

## Integration Notes for Your Backend Projects
- Place `web/` at project root (exactly like atstex-lab).
- Add `web.InitTemplates()` to `cmd/server/main.go` (after logger, before router).
- Combine with `chi-router-and-middleware-best-practices` → add static route + page routes.
- Combine with `embedded-static-files-and-react-spa-serving` → use both: `/static/` for assets + SPA for React if needed.
- Combine with `graceful-shutdown-with-signal-handling` → no changes (templates are in-memory).
- For atstex-lab-style projects: the `web/templates` + `embed.go` pattern is already used for the CV builder UI.

## Makefile Targets (recommended)

```makefile
.PHONY: templates
templates: ## Re-embed templates (run after editing)
	@echo "Templates are embedded at build time - no action needed"
```

## Why This Pattern Wins
- Tiny binary (everything embedded)
- Fast rendering (pre-parsed)
- Secure (html/template + escaping)
- Maintainable (clear layouts/partials/pages split)
- Consistent with SemmiDev/atstex-lab web structure

Follow this skill and every server-side HTML page in your Go backends will be:
- Clean and DRY
- Secure and performant
- Easy to theme/customize
- Production-ready out of the box

When in doubt, copy the `web/embed.go` + `base.html` + `Render` helper pattern above — this is the exact design used across SemmiDev backend projects (including atstex-lab).

You now have a complete, professional server-side HTML foundation! Ready for the next skill. 🚀
