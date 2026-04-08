---
name: secure-random-generator
description: Idiomatic, cryptographically-secure random data generation for Go backend projects. Extends the existing internal/common/random package (uuid.go) with fast, zero-dependency helpers using only crypto/rand (stdlib). Provides RandomString, RandomAlphanumeric, GenerateOTP (numeric), RandomBytes, and secure token helpers. Never use math/rand for security-sensitive values. Perfect for OTP codes, reset tokens, API keys, session IDs, etc. Integrates with clean-idiomatic-config-loading, rich-domain-model, and all previous skills.
---

# Secure Random Generator

**Skill purpose**  
This skill extends the existing `internal/common/random` package (already containing UUID v7) with **production-grade, cryptographically secure** random generators.  

It follows 2026 Go best practices:
- **Always** use `crypto/rand` (OS-backed entropy) — never `math/rand`.
- Zero external dependencies for core functions (keeps binary small and secure).
- Efficient, allocation-friendly implementation (byte slicing + charset mapping).
- Focused helpers for the most common use cases: OTP, alphanumeric tokens, API keys, etc.
- Secure by default (high entropy, constant-time where possible).

This matches the lightweight, in-house style of SemmiDev projects while being as secure as popular libraries like `go-random` or `randstr`.

## When to use this skill
- Generate one-time passwords (OTP) for email/SMS login.
- Create secure API keys, reset tokens, session tokens, invite codes.
- Generate random alphanumeric strings (passwords, filenames, etc.).
- Any place where `crypto/rand` is needed but you want a clean helper.

**Never**:
- Use `math/rand` for anything security-related.
- Implement your own random logic in handlers/services.
- Rely on predictable sources (time-based seeds, etc.).

## Project Structure (extend existing package)

```
internal/common/random/
├── uuid.go        ← (existing)
├── random.go      ← ←← NEW FILE (this skill)
└── random_test.go
```

## Canonical Implementation (`random.go`)

```go
package random

import (
    "crypto/rand"
    "encoding/base64"
    "fmt"
    "math/big"
)

// charset constants (define once, reuse everywhere)
const (
    Alphanumeric = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
    Numeric      = "0123456789"
    Hex          = "0123456789abcdef"
    Readable     = "23456789abcdefghjkmnpqrstuvwxyz" // avoids confusing chars (0/O, 1/I/l)
)

// RandomBytes returns cryptographically secure random bytes.
func RandomBytes(length int) ([]byte, error) {
    b := make([]byte, length)
    _, err := rand.Read(b)
    if err != nil {
        return nil, fmt.Errorf("crypto/rand.Read failed: %w", err)
    }
    return b, nil
}

// RandomString generates a secure random string from a custom charset.
func RandomString(length int, charset string) (string, error) {
    if length <= 0 {
        return "", fmt.Errorf("length must be > 0")
    }
    if charset == "" {
        charset = Alphanumeric
    }

    bytes := make([]byte, length)
    charsetLen := big.NewInt(int64(len(charset)))

    for i := 0; i < length; i++ {
        idx, err := rand.Int(rand.Reader, charsetLen)
        if err != nil {
            return "", fmt.Errorf("failed to generate random index: %w", err)
        }
        bytes[i] = charset[idx.Int64()]
    }
    return string(bytes), nil
}

// RandomAlphanumeric is the most common helper (base64-like but readable).
func RandomAlphanumeric(length int) (string, error) {
    return RandomString(length, Alphanumeric)
}

// GenerateOTP returns a secure numeric OTP (e.g. 6 digits for SMS/email).
func GenerateOTP(digits int) (string, error) {
    if digits < 4 || digits > 10 {
        return "", fmt.Errorf("OTP digits must be between 4 and 10")
    }
    return RandomString(digits, Numeric)
}

// RandomToken returns a secure base64 URL-safe token (common for API keys / sessions).
func RandomToken(length int) (string, error) {
    b, err := RandomBytes(length)
    if err != nil {
        return "", err
    }
    return base64.RawURLEncoding.EncodeToString(b), nil
}

// Must* variants (for init/tests – panic on error)
func MustRandomString(length int, charset string) string {
    s, err := RandomString(length, charset)
    if err != nil {
        panic(err)
    }
    return s
}

func MustRandomAlphanumeric(length int) string {
    return MustRandomString(length, Alphanumeric)
}

func MustGenerateOTP(digits int) string {
    otp, err := GenerateOTP(digits)
    if err != nil {
        panic(err)
    }
    return otp
}
```

## Recommended Usage Patterns

### 1. OTP for Email/SMS Verification
```go
otp, err := random.GenerateOTP(6) // "482193"
if err != nil { /* handle */ }

// Store in DB with expiry (use your repository + TTL)
err = repo.StoreOTP(ctx, email, otp, 10*time.Minute)
```

### 2. Secure API Key / Reset Token
```go
token, err := random.RandomToken(32) // ~43 chars base64 URL-safe
// or
key, err := random.RandomAlphanumeric(32)
```

### 3. In Domain Layer (rich domain example)
```go
func (u *User) GeneratePasswordResetToken() (string, error) {
    return random.RandomToken(32)
}
```

### 4. In Tests
```go
token := random.MustRandomAlphanumeric(16)
```

## Best Practices & Decision Tree (2026 consensus)

**Always**:
- Use `crypto/rand` (or the helpers above) for anything security-related.
- Store OTPs/tokens with short expiry + rate limiting.
- Use `Readable` charset for user-facing codes (avoids confusion).
- Log only hashes of tokens/OTPs, never the raw value.

**Decision Tree**:
1. Need 6-digit code for SMS/email? → `GenerateOTP(6)`
2. Need long secure token (API key, session)? → `RandomToken(32)` or `RandomAlphanumeric(32)`
3. Need custom charset? → `RandomString(length, charset)`
4. Need TOTP (Google Authenticator style)? → Add `github.com/pquerna/otp` as optional dependency in a separate `totp.go` (not in core random package).

**Never**:
- Use `math/rand` for OTPs, tokens, keys.
- Generate random data in hot paths without error checking.
- Hardcode charset in multiple places — use the constants.

## Integration Notes for Your Backend Projects
- Add `random.go` to the existing `internal/common/random/` package (next to `uuid.go`).
- Update `go.mod` only if you later add TOTP (`go get github.com/pquerna/otp`).
- Combine with `secure-idiomatic-error-handling` for OTP storage errors.
- Combine with `structured-logging-with-slog` (log "OTP generated" without the code).
- Combine with `database-access-with-pgx-and-sqlx` (store hashed OTP + expiry).

Follow this skill and every random value in your Go backends will be:
- Cryptographically secure
- Consistent
- Zero-dependency (core functions)
- Easy to audit and test

When in doubt, use `random.GenerateOTP(6)` or `random.RandomAlphanumeric(32)` — this is the exact secure, idiomatic pattern now used across all SemmiDev backend projects.
