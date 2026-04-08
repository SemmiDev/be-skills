```markdown
---
name: smtp-email-sender-abstraction
description: Idiomatic, testable SMTP email sender for Go backend projects using github.com/wneessen/go-mail (modern, context-aware, full-featured SMTP library with HTML/attachments/template support). Provides a clean Sender interface + Config for dependency injection and mocking. Perfect integration with http-error-handling-with-problem skill and validation. Always use this abstraction instead of raw net/smtp or direct library calls in handlers/services.
---

# SMTP Email Sender Abstraction (Idiomatic Go)

**Skill purpose**  
This skill defines the **official SemmiDev pattern** for sending emails via SMTP in all Go backends.  

It uses:
- A minimal `Sender` interface for easy dependency injection and unit testing (no real SMTP in tests).
- A production-grade implementation wrapping `github.com/wneessen/go-mail` (the current best-in-class SMTP library in 2026: context support, TLS/STARTTLS, attachments, HTML+plain multipart, templates, middleware-ready).
- Config struct driven by environment variables.
- Automatic error wrapping into RFC 7807 Problem Details.
- Support for plain text, HTML (with fallback), attachments, and future template rendering.

This keeps your handlers/services clean, your tests fast, and your email layer swappable (e.g. to AWS SES or SendGrid API later).

## When to use this skill
- Any transactional email (welcome, reset password, order confirmation, notifications, etc.).
- You need HTML emails with plain-text fallback and attachments.
- You want context cancellation/timeouts on email sending.
- You are replacing raw `net/smtp.SendMail` or direct library usage.
- You need to mock email sending in unit/integration tests.

**Never** call SMTP libraries directly from handlers or business logic.

## Project Structure (recommended)

```
internal/common/email/
├── email.go       ← interface + types + Message
├── smtp.go        ← SMTP implementation using go-mail
└── errors.go      ← optional custom error wrapping
```

## Installation

```bash
go get github.com/wneessen/go-mail
```

## Core Concepts

### 1. Sender Interface & Types (`email.go`)

```go
package email

import (
    "context"
    "io"
)

type Sender interface {
    // Send a single email. Returns error wrapped for problem library if needed.
    Send(ctx context.Context, msg Message) error
}

type Message struct {
    From        string            // e.g. "App Name <no-reply@yourdomain.com>"
    To          []string          // required
    CC          []string          `json:"cc,omitempty"`
    BCC         []string          `json:"bcc,omitempty"`
    Subject     string
    TextBody    string            // plain text fallback (recommended)
    HTMLBody    string            // HTML version
    Attachments []Attachment
}

type Attachment struct {
    Filename    string
    Content     io.Reader     // or use File path helper in impl
    ContentType string        // optional, auto-detected
}

type Config struct {
    Host     string
    Port     int
    Username string
    Password string
    From     string // default From address

    // TLS/STARTTLS policy (use Mandatory for production)
    UseTLS         bool // true = TLSMandatory + STARTTLS on 587
    TimeoutSeconds int  // default 15
}
```

### 2. SMTP Implementation (`smtp.go`)

```go
package email

import (
    "context"
    "fmt"
    "time"

    "github.com/wneessen/go-mail"
    problem "github.com/semmidev/problem"
)

type smtpSender struct {
    client *mail.Client
    from   string
}

func NewSMTP(cfg Config) (Sender, error) {
    opts := []mail.Option{
        mail.WithPort(cfg.Port), // usually 587
        mail.WithTimeout(time.Duration(cfg.TimeoutSeconds)*time.Second),
    }

    if cfg.UseTLS {
        opts = append(opts,
            mail.WithTLSPolicy(mail.TLSMandatory),
            mail.WithSMTPAuth(mail.SMTPAuthPlain),
            mail.WithUsername(cfg.Username),
            mail.WithPassword(cfg.Password),
        )
    } else {
        // Plain or Opportunistic – not recommended for production
        opts = append(opts, mail.WithTLSPolicy(mail.TLSOpportunistic))
    }

    client, err := mail.NewClient(cfg.Host, opts...)
    if err != nil {
        return nil, fmt.Errorf("failed to create SMTP client: %w", err)
    }

    return &smtpSender{
        client: client,
        from:   cfg.From,
    }, nil
}

func (s *smtpSender) Send(ctx context.Context, msg Message) error {
    m := mail.NewMsg()

    // From
    if err := m.From(msg.From); err != nil && msg.From == "" {
        if err := m.From(s.from); err != nil {
            return s.wrapError(err, "invalid from address")
        }
    } else if err != nil {
        return s.wrapError(err, "invalid from address")
    }

    // Recipients
    if err := m.To(msg.To...); err != nil {
        return s.wrapError(err, "invalid to addresses")
    }
    if len(msg.CC) > 0 {
        m.Cc(msg.CC...)
    }
    if len(msg.BCC) > 0 {
        m.Bcc(msg.BCC...)
    }

    m.Subject(msg.Subject)

    // Bodies (HTML + plain fallback)
    if msg.HTMLBody != "" {
        m.SetBodyString(mail.TypeTextHTML, msg.HTMLBody)
    }
    if msg.TextBody != "" {
        m.SetBodyString(mail.TypeTextPlain, msg.TextBody)
    } else if msg.HTMLBody != "" {
        // Auto-generate plain text fallback if only HTML is provided (optional helper)
        m.SetBodyString(mail.TypeTextPlain, stripHTML(msg.HTMLBody))
    }

    // Attachments
    for _, att := range msg.Attachments {
        if err := m.AttachReader(att.Filename, att.Content, mail.WithFileContentType(att.ContentType)); err != nil {
            return s.wrapError(err, "failed to attach file")
        }
    }

    // Send with context
    if err := s.client.DialAndSendWithContext(ctx, m); err != nil {
        return s.wrapError(err, "failed to send email")
    }
    return nil
}

func (s *smtpSender) wrapError(err error, op string) error {
    // Wrap for problem library compatibility
    return fmt.Errorf("smtp %s: %w", op, err)
    // In handlers you can check with errors.Is / problem.IsProblem
}
```

*(Add helper `stripHTML` using `github.com/aymerick/douceur` or simple regex if needed – optional.)*

## Recommended Usage Patterns

### 1. Setup (in main or DI)
```go
var EmailSender email.Sender

func initEmail() {
    cfg := email.Config{
        Host:           os.Getenv("SMTP_HOST"),
        Port:           587,
        Username:       os.Getenv("SMTP_USERNAME"),
        Password:       os.Getenv("SMTP_PASSWORD"),
        From:           os.Getenv("SMTP_FROM"),
        UseTLS:         true,
        TimeoutSeconds: 15,
    }
    var err error
    EmailSender, err = email.NewSMTP(cfg)
    if err != nil {
        panic(err)
    }
}
```

### 2. Handler Example (with problem integration)
```go
func SendWelcomeEmail(w http.ResponseWriter, r *http.Request) {
    var req struct {
        Email string `json:"email" validate:"required,email"`
        Name  string `json:"name"`
    }
    // decode + validate with ethos validator...

    msg := email.Message{
        To:       []string{req.Email},
        Subject:  "Welcome to Our App!",
        TextBody: fmt.Sprintf("Hi %s, welcome!", req.Name),
        HTMLBody: fmt.Sprintf(`<h1>Welcome %s!</h1><p>Thank you for joining.</p>`, req.Name),
    }

    if err := EmailSender.Send(r.Context(), msg); err != nil {
        problem.New(problem.InternalServerError,
            problem.WithDetail("Failed to send welcome email"),
            problem.WithExtension("original_error", err.Error()),
        ).Write(w)
        return
    }

    // success response
}
```

### 3. Testing (mock the interface)
```go
type mockSender struct{}

func (mockSender) Send(ctx context.Context, msg email.Message) error {
    // record call for assertion
    return nil
}

func TestWelcomeFlow(t *testing.T) {
    EmailSender = mockSender{}
    // call handler or service
}
```

## Best Practices & Decision Tree

**Always**:
- Use `UseTLS: true` + port 587 (STARTTLS).
- Always provide both `TextBody` and `HTMLBody`.
- Pass `context.Context` from the request.
- Use environment variables for credentials (never commit them).
- Wrap errors with the problem skill.

**Decision Tree**:
1. Simple transactional email? → `Sender.Send(...)`
2. Need attachments? → Add to `Message.Attachments`
3. Want templates? → Add `HTMLTemplate`/`TextTemplate` method to Message (easy extension using go-mail’s built-in template support).
4. High volume? → Later add connection pooling via go-mail’s reuse or switch to SES API.
5. Testing? → Always mock the `Sender` interface.

**Never**:
- Hardcode SMTP credentials.
- Send emails without context timeout.
- Use `net/smtp` directly.
- Return raw SMTP errors to the client (always wrap via problem).

## Suggested Improvements (add directly)

Add to `email.go`:
```go
// Template support (recommended)
func (m *Message) WithHTMLTemplate(templateName string, data any) {
    // render using html/template + embed.FS in your project
    // then set m.HTMLBody
}
```

Or add bulk send method to interface if needed.

## Integration Notes for Your Backend Projects
- Place the package in `internal/common/email`.
- Combine with validation skill (validate email addresses before sending).
- Combine with problem skill for consistent error responses.
- For local/dev: use MailHog or Mailtrap SMTP (set `UseTLS: false` + their credentials).
- Document your default From address and common templates in README.
- Monitor bounce rates / delivery via your SMTP provider’s dashboard.

Follow this skill and every email in your Go backends will be:
- Clean & idiomatic
- Fully testable
- Production-ready (TLS, context, multipart, attachments)
- Future-proof (easy to swap implementation)

When in doubt, copy the `Config` + `Message` + `Send` pattern above — this is the exact design used across SemmiDev backend projects.  

Combine with validation + http-error + storage skills = complete, professional backend foundations.
