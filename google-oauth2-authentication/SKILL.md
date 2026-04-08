---
name: google-oauth2-authentication
description: Production-grade Google OAuth2 authentication for Go backends using golang.org/x/oauth2 + google.Endpoint. Directly based on the exact pattern from SemmiDev/atstex-lab/internal/auth/auth.go (OAuthConfig setup, state cookie for CSRF, session_token cookie, middleware loading User into context). Provides login redirect, callback handler (code exchange + userinfo + user creation/linking + session), and reusable middleware. Integrates with rich-domain-model-design-ddd (domain.User), clean-idiomatic-config-loading, chi-router-and-middleware-best-practices, and http-error-handling-with-problem. Always use this for Google login instead of manual OAuth handling.
---

# Google OAuth2 Authentication

**Skill purpose**  
This skill defines the **official SemmiDev pattern** for Google OAuth2 login, taken directly from `atstex-lab/internal/auth/auth.go` and enhanced with 2026 best practices (state cookie CSRF protection, secure session cookies, domain.User integration, and Chi middleware).  

It implements the full **Authorization Code Flow**:
- `/auth/google` → redirect to Google with state cookie
- `/auth/google/callback` → exchange code → fetch userinfo → find/create User → issue session cookie → redirect to frontend

The pattern is **cookie-based sessions** (not JWT by default), matching atstex-lab’s server-rendered/hybrid style. It works perfectly with React SPA (redirect back to `/` with session cookie) or pure HTML pages.

## When to use this skill
- Google “Sign in with Google” button for web apps.
- You need secure session management with domain.User in context.
- You want CSRF protection via state cookie (as in atstex-lab).
- You are adding social login to an existing API + SPA setup.

**Never** implement raw OAuth redirects or token exchange manually.

## Installation

```bash
go get golang.org/x/oauth2
go get golang.org/x/oauth2/google
```

(Your `config.Load()` already provides `GoogleClientID`, `GoogleClientSecret`, and `GoogleCallbackURL`.)

## Project Structure

```
internal/auth/
├── auth.go          ← Core Google auth (exact atstex-lab style + extensions)
├── middleware.go    ← Session + Admin + OptionalUser middleware
└── user.go          ← Google profile → domain.User mapping
```

## Core Implementation

### 1. `internal/auth/auth.go` (canonical Google setup)

```go
package auth

import (
    "context"
    "crypto/rand"
    "encoding/base64"
    "net/http"
    "time"

    "github.com/semmidev/problem" // your error skill
    "golang.org/x/oauth2"
    "golang.org/x/oauth2/google"
    "yourproject/internal/config"
    "yourproject/internal/domain" // rich domain.User
    "yourproject/internal/repository"
)

type GoogleAuth struct {
    oauthConfig *oauth2.Config
    repo        repository.Repository
}

func NewGoogleAuth(cfg *config.AppConfig, repo repository.Repository) *GoogleAuth {
    return &GoogleAuth{
        oauthConfig: &oauth2.Config{
            ClientID:     cfg.GoogleClientID,
            ClientSecret: cfg.GoogleClientSecret,
            RedirectURL:  cfg.GoogleCallbackURL,
            Scopes: []string{
                "https://www.googleapis.com/auth/userinfo.email",
                "https://www.googleapis.com/auth/userinfo.profile",
                "openid",
            },
            Endpoint: google.Endpoint,
        },
        repo: repo,
    }
}

// LoginHandler redirects to Google with CSRF state cookie (exact atstex-lab pattern)
func (g *GoogleAuth) LoginHandler() http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        state := generateStateOauthCookie(w)
        url := g.oauthConfig.AuthCodeURL(state)
        http.Redirect(w, r, url, http.StatusTemporaryRedirect)
    }
}

// CallbackHandler exchanges code, fetches userinfo, creates/finds user, sets session cookie
func (g *GoogleAuth) CallbackHandler() http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        state := r.FormValue("state")
        cookie, err := r.Cookie("oauthstate")
        if err != nil || cookie.Value != state {
            problem.New(problem.BadRequest, problem.WithDetail("Invalid OAuth state")).Write(w)
            return
        }

        code := r.FormValue("code")
        token, err := g.oauthConfig.Exchange(r.Context(), code)
        if err != nil {
            problem.New(problem.InternalServerError, problem.WithDetail("Failed to exchange token")).Write(w)
            return
        }

        // Fetch Google user info
        client := g.oauthConfig.Client(r.Context(), token)
        resp, err := client.Get("https://www.googleapis.com/oauth2/v2/userinfo")
        if err != nil {
            problem.New(problem.InternalServerError).Write(w)
            return
        }
        defer resp.Body.Close()

        // Parse userinfo → map to domain.User (implement GoogleUserToDomain in user.go)
        googleUser, err := parseGoogleUserInfo(resp.Body)
        if err != nil {
            problem.New(problem.InternalServerError).Write(w)
            return
        }

        // Find or create user in domain + repo
        user, err := g.repo.FindOrCreateGoogleUser(r.Context(), googleUser)
        if err != nil {
            problem.New(problem.InternalServerError).Write(w)
            return
        }

        // Create session token and store in repo
        sessionToken, err := generateSessionToken()
        if err != nil {
            problem.New(problem.InternalServerError).Write(w)
            return
        }

        if err := g.repo.CreateSession(r.Context(), sessionToken, user.ID, time.Now().Add(30*24*time.Hour)); err != nil {
            problem.New(problem.InternalServerError).Write(w)
            return
        }

        // Set secure session cookie
        setSessionCookie(w, sessionToken)

        // Redirect to frontend (SPA or HTML page)
        http.Redirect(w, r, "/", http.StatusSeeOther)
    }
}

func generateStateOauthCookie(w http.ResponseWriter) string {
    b := make([]byte, 16)
    _, _ = rand.Read(b)
    state := base64.URLEncoding.EncodeToString(b)
    cookie := &http.Cookie{
        Name:     "oauthstate",
        Value:    state,
        Expires:  time.Now().Add(10 * time.Minute),
        HttpOnly: true,
        Secure:   true,
        Path:     "/",
        SameSite: http.SameSiteLaxMode,
    }
    http.SetCookie(w, cookie)
    return state
}

func generateSessionToken() (string, error) {
    b := make([]byte, 32)
    _, err := rand.Read(b)
    if err != nil {
        return "", err
    }
    return base64.URLEncoding.EncodeToString(b), nil
}

func setSessionCookie(w http.ResponseWriter, token string) {
    cookie := &http.Cookie{
        Name:     "session_token",
        Value:    token,
        Expires:  time.Now().Add(30 * 24 * time.Hour),
        HttpOnly: true,
        Secure:   true,
        Path:     "/",
        SameSite: http.SameSiteLaxMode,
    }
    http.SetCookie(w, cookie)
}
```

### 2. Middleware (`internal/auth/middleware.go`) – exact atstex-lab style

```go
package auth

import (
    "context"
    "net/http"
    "time"

    "yourproject/internal/domain"
    "yourproject/internal/repository"
)

// Context key
type contextKey string
const UserContextKey contextKey = "user"

// Middleware loads authenticated user from session cookie into context
func Middleware(repo repository.Repository) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            cookie, err := r.Cookie("session_token")
            if err != nil {
                next.ServeHTTP(w, r) // optional user middleware variant exists
                return
            }

            sess, err := repo.GetSession(r.Context(), cookie.Value)
            if err != nil || sess.ExpiresAt.Before(time.Now()) {
                clearCookie(w)
                next.ServeHTTP(w, r)
                return
            }

            u, err := repo.GetUser(r.Context(), sess.UserID)
            if err != nil {
                clearCookie(w)
                next.ServeHTTP(w, r)
                return
            }

            ctx := context.WithValue(r.Context(), UserContextKey, u)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}

// AdminMiddleware (after regular Middleware)
func AdminMiddleware() func(http.Handler) http.Handler { ... } // as in atstex-lab

func clearCookie(w http.ResponseWriter) { ... } // as in atstex-lab
```

### 3. Google Profile → Domain User (`internal/auth/user.go`)

```go
type GoogleUserInfo struct {
    ID            string `json:"id"`
    Email         string `json:"email"`
    VerifiedEmail bool   `json:"verified_email"`
    Name          string `json:"name"`
    Picture       string `json:"picture"`
}

func (g *GoogleAuth) FindOrCreateGoogleUser(ctx context.Context, info GoogleUserInfo) (*domain.User, error) {
    // Use rich domain factory + repo
    return g.repo.FindOrCreateFromGoogle(ctx, info)
}
```

## Router Integration (Chi)

```go
func (r *chi.Mux) MountAuthRoutes(auth *auth.GoogleAuth) {
    r.Route("/auth", func(r chi.Router) {
        r.Get("/google", auth.LoginHandler())
        r.Get("/google/callback", auth.CallbackHandler())
    })
}
```

## Best Practices & Decision Tree

**Always**:
- Use state cookie (exact atstex-lab pattern) for CSRF.
- Use `HttpOnly + Secure + SameSiteLax` cookies.
- Store session in DB (repo.CreateSession / GetSession).
- Map Google userinfo to rich `domain.User`.
- Call `auth.Middleware` early in middleware stack.

**Decision Tree**:
1. Web app with cookies? → Use this skill (session cookies).
2. Pure API + SPA with JWT? → Extend with JWT issuance on callback.
3. Need PKCE? → Add code_verifier (future-proof extension).

**Never**:
- Hardcode client credentials.
- Skip state validation.
- Store sensitive tokens in logs.

## Integration Notes
- Place in `internal/auth/` (exact atstex-lab location).
- Use your `config.AppConfig` + `repository.Repository`.
- Mount after Chi middleware stack but before protected routes.
- Combine with `rich-domain-model-design-ddd` (domain.User).
- Combine with `clean-idiomatic-config-loading` (Google credentials).
- Combine with `chi-router-and-middleware-best-practices`.

Follow this skill and your Google auth will be:
- Secure (CSRF + secure cookies)
- Consistent with atstex-lab
- Easy to extend (GitHub, etc.)

When in doubt, copy the `NewGoogleAuth` + handlers + middleware pattern above — this is the exact production pattern used in all SemmiDev backend projects.
