```markdown
---
name: s3-storage-abstraction
description: Idiomatic, testable S3 (and S3-compatible) storage layer for Go backend projects using a clean interface + wrapper pattern. Built on AWS SDK for Go v2. Provides Upload/Download/Delete/Presigned URLs + large-file support via manager. Perfectly integrates with http-error-handling-with-problem and validation skills. Always use this abstraction instead of raw *s3.Client calls.
---

# S3 Storage Abstraction (Idiomatic Go Interface)

**Skill purpose**  
This skill defines the **official SemmiDev pattern** for handling Amazon S3 (and any S3-compatible storage like MinIO, R2, Wasabi) in all Go backends.  

It uses:
- A clean `Storage` interface for dependency injection and easy mocking in tests.
- A production-ready `s3Storage` implementation wrapping `*s3.Client` + `*s3.PresignClient`.
- Config struct for environment-driven setup.
- Built-in support for large files (multipart via `s3/manager`).
- Presigned URLs (GET/PUT/POST) for secure direct client uploads/downloads.
- Proper context propagation, metadata, content-type, and error wrapping.

This keeps your handlers clean, your code testable, and your storage layer swappable (local FS in dev if you want later).

## When to use this skill
- Any file upload/download in your HTTP handlers (images, documents, videos, exports).
- Generating presigned URLs for frontend direct-to-S3 uploads (recommended for large files / high traffic).
- Storing/retrieving user-generated content, backups, media, etc.
- You need consistent error handling that integrates with `github.com/semmidev/problem`.
- You want to unit-test storage logic without hitting real S3.

**Never** use raw `s3.New(...)` or `s3.Client` directly in handlers or services.

## Project Structure (recommended)

```
internal/common/storage/
├── storage.go      ← interface + types
├── s3.go           ← S3 implementation + config
└── errors.go       ← optional custom error types (if needed)
```

## Installation

```bash
go get github.com/aws/aws-sdk-go-v2
go get github.com/aws/aws-sdk-go-v2/config
go get github.com/aws/aws-sdk-go-v2/service/s3
go get github.com/aws/aws-sdk-go-v2/feature/s3/manager
go get github.com/aws/aws-sdk-go-v2/aws
```

## Core Concepts

### 1. Storage Interface (`storage.go`)

```go
package storage

import (
    "context"
    "io"
    "time"
)

type Storage interface {
    // Upload file (small or large)
    Upload(ctx context.Context, key string, body io.Reader, opts UploadOptions) error

    // Download returns ReadCloser (caller must Close!)
    Download(ctx context.Context, key string) (io.ReadCloser, error)

    // Delete single object
    Delete(ctx context.Context, key string) error

    // Presigned GET URL (for downloading)
    PresignGet(ctx context.Context, key string, expires time.Duration) (string, error)

    // Presigned PUT URL (for direct client upload)
    PresignPut(ctx context.Context, key string, expires time.Duration, opts UploadOptions) (string, error)

    // Optional: Head / List / DeleteMany if needed
    Head(ctx context.Context, key string) (*ObjectInfo, error)
}

type UploadOptions struct {
    ContentType string            // e.g. "image/jpeg"
    Metadata    map[string]string // custom x-amz-meta-*
    ACL         string            // "private", "public-read" (default private)
}

type ObjectInfo struct {
    Key          string
    Size         int64
    ContentType  string
    LastModified time.Time
    Metadata     map[string]string
}
```

### 2. S3 Implementation + Config (`s3.go`)

```go
package storage

import (
    "context"
    "fmt"
    "io"
    "time"

    "github.com/aws/aws-sdk-go-v2/aws"
    "github.com/aws/aws-sdk-go-v2/config"
    "github.com/aws/aws-sdk-go-v2/feature/s3/manager"
    "github.com/aws/aws-sdk-go-v2/service/s3"
    "github.com/aws/aws-sdk-go-v2/service/s3/types"
    "github.com/semmidev/problem" // for error wrapping (optional)
)

type S3Config struct {
    Bucket         string
    Region         string
    Endpoint       string // for MinIO / local S3-compatible (optional)
    UsePathStyle   bool   // true for MinIO, false for AWS
    AccessKeyID    string // if not using IAM roles / env
    SecretAccessKey string
}

type s3Storage struct {
    client    *s3.Client
    presigner *s3.PresignClient
    uploader  *manager.Uploader
    downloader *manager.Downloader
    bucket    string
}

func NewS3Storage(cfg S3Config) (Storage, error) {
    awsCfg, err := config.LoadDefaultConfig(context.Background(),
        config.WithRegion(cfg.Region),
        // If using static credentials (for MinIO or testing)
        config.WithCredentialsProvider(aws.CredentialsProviderFunc(func(ctx context.Context) (aws.Credentials, error) {
            if cfg.AccessKeyID == "" {
                return aws.Credentials{}, nil // use default chain
            }
            return aws.Credentials{
                AccessKeyID:     cfg.AccessKeyID,
                SecretAccessKey: cfg.SecretAccessKey,
            }, nil
        })),
    )
    if err != nil {
        return nil, fmt.Errorf("failed to load AWS config: %w", err)
    }

    opts := []func(*s3.Options){}
    if cfg.Endpoint != "" {
        opts = append(opts, func(o *s3.Options) {
            o.BaseEndpoint = aws.String(cfg.Endpoint)
            o.UsePathStyle = cfg.UsePathStyle
        })
    }

    client := s3.NewFromConfig(awsCfg, opts...)
    presigner := s3.NewPresignClient(client)

    return &s3Storage{
        client:     client,
        presigner:  presigner,
        uploader:   manager.NewUploader(client),
        downloader: manager.NewDownloader(client),
        bucket:     cfg.Bucket,
    }, nil
}

// Example Upload implementation (uses manager for large files automatically)
func (s *s3Storage) Upload(ctx context.Context, key string, body io.Reader, opts UploadOptions) error {
    input := &s3.PutObjectInput{
        Bucket:      aws.String(s.bucket),
        Key:         aws.String(key),
        Body:        body,
        ContentType: aws.String(opts.ContentType),
        Metadata:    opts.Metadata,
    }

    if opts.ACL != "" {
        input.ACL = types.ObjectCannedACL(opts.ACL)
    }

    _, err := s.uploader.Upload(ctx, input)
    if err != nil {
        return s.wrapError(err, "upload", key)
    }
    return nil
}

// Download, Delete, PresignGet, PresignPut follow the same clean pattern.
// (Full implementations are in the official AWS examples + manager.)

func (s *s3Storage) wrapError(err error, op, key string) error {
    // Optional: convert AWS errors into problem.Problem for consistency
    // Example:
    // if apiErr, ok := err.(smithy.APIError); ok { ... }
    return fmt.Errorf("s3 %s %s: %w", op, key, err)
}
```

*(Copy the full `PresignGet`, `PresignPut`, `Download`, `Delete`, `Head` methods from AWS official examples — they are short and production-ready.)*

## Recommended Usage Patterns

### 1. Setup (in main or DI container)
```go
var S3 Storage

func initStorage() {
    cfg := storage.S3Config{
        Bucket:   os.Getenv("S3_BUCKET"),
        Region:   os.Getenv("AWS_REGION"),
        Endpoint: os.Getenv("S3_ENDPOINT"), // empty for AWS
    }
    var err error
    S3, err = storage.NewS3Storage(cfg)
    if err != nil {
        panic(err) // or handle gracefully
    }
}
```

### 2. Handler with direct upload (server-side)
```go
func UploadProfilePicture(w http.ResponseWriter, r *http.Request) {
    file, header, err := r.FormFile("file")
    if err != nil { /* handle */ }

    key := fmt.Sprintf("users/%s/profile/%s", userID, header.Filename)

    if err := S3.Upload(r.Context(), key, file, storage.UploadOptions{
        ContentType: header.Header.Get("Content-Type"),
    }); err != nil {
        problem.New(problem.InternalServerError,
            problem.WithDetail("Failed to upload file"),
        ).Write(w)
        return
    }

    // success
}
```

### 3. Presigned URL for frontend direct upload (recommended for large files)
```go
func GetUploadURL(w http.ResponseWriter, r *http.Request) {
    var req struct { Filename string `json:"filename"` }
    // decode...

    key := fmt.Sprintf("uploads/%s/%s", userID, req.Filename)
    url, err := S3.PresignPut(r.Context(), key, 15*time.Minute, storage.UploadOptions{
        ContentType: "image/jpeg",
    })
    if err != nil { /* problem */ }

    // return { "url": url, "key": key }
}
```

### 4. Testing (mock the interface!)
```go
type mockStorage struct{ /* implement Storage */ }

func TestUploadHandler(t *testing.T) {
    storageMock := &mockStorage{}
    // inject into handler
}
```

## Best Practices & Decision Tree

**Always**:
- Pass `context.Context` everywhere.
- Use the `Storage` interface in services/handlers.
- Prefer presigned URLs for client-side uploads (reduces your server load).
- Use `manager.Uploader`/`Downloader` for any file > 5MB.
- Set proper Content-Type and Metadata.

**Decision Tree**:
1. Small file, server receives it? → `Upload(...)`
2. Large file or frontend upload? → Generate presigned PUT URL
3. Need temporary download link? → `PresignGet(...)`
4. Need to test? → Mock the `Storage` interface

**Never**:
- Hardcode AWS credentials in code.
- Call `s3.Client` directly outside the abstraction.
- Forget to close `io.ReadCloser` from `Download`.

## Integration Notes for Your Backend Projects
- Put the `storage` package in `internal/common/storage`.
- Use environment variables for config (or your existing config system).
- Combine with the `http-error-handling-with-problem` skill: wrap S3 errors into `problem.InternalServerError` or custom types.
- For local dev: use MinIO + set `Endpoint` and `UsePathStyle: true`.
- Document your bucket naming convention and folder structure in your project README.

Follow this skill and every storage operation in your Go backends will be:
- Clean & idiomatic
- Fully testable
- Production-ready (multipart, presigned, context-aware)
- Future-proof (easy to add local FS or another provider)

When in doubt, follow the interface + `NewS3Storage` pattern above — this is the exact design used across SemmiDev backend projects.  

Combine with validation + problem skills = complete, professional file handling.
