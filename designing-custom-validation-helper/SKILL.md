```markdown
---
name: designing-custom-validation-helper
description: Canonical technique and file structure for building a reusable, localized validation helper. Inspired directly by the design patterns in ethos-go/internal/common/validator (message.go + rules.go + validator.go). This skill shows exactly how to design, structure, and implement your own internal validation package — no external library dependency beyond go-playground/validator/v10. It guarantees consistent field-level errors, full English/Indonesian localization, Indonesian-specific rules, and perfect integration with the http-error-handling-with-problem skill (always returns 422 + field_errors extension).
---

# Designing a Custom Validation Helper (Project Technique)

**Skill purpose**  
This skill teaches the exact **design technique** used in SemmiDev projects for a clean, maintainable, and extensible validation helper.  
You do **not** import an external validator package. Instead, you copy and own this small internal package (`internal/common/validator`) in every backend project.  

The technique gives you:
- Locale-aware error messages (EN/ID)
- Easy-to-extend custom rules (phone_id, nik, strong_password, etc.)
- Beautiful JSON error format ready for RFC 7807 Problem Details
- Zero boilerplate in handlers
- Full compatibility with `github.com/semmidev/problem`

**Why this design?**  
- Single source of truth for all validation logic  
- Easy to add new rules or languages  
- Field names come from `json` tags automatically  
- Errors are structured as `[]ValidationError` → perfect for `problem.WithExtension("field_errors", errs.ToKV())`

## Project Structure (copy this exactly)

```
internal/common/validator/
├── message.go     ← localized messages (EN/ID)
├── rules.go       ← all custom validation functions
└── validator.go   ← core Validator struct + helpers
```

## 1. validator.go (Core Design)

```go
package validator

import (
	"fmt"
	"reflect"
	"strings"

	"github.com/go-playground/validator/v10"
)

type Validator struct {
	validate *validator.Validate
	locale   string
}

func New(locale string) *Validator {
	validate := validator.New()

	// Use json tag for field names in errors (best practice)
	validate.RegisterTagNameFunc(func(fld reflect.StructField) string {
		name := strings.SplitN(fld.Tag.Get("json"), ",", 2)[0]
		if name == "-" {
			return ""
		}
		return name
	})

	v := &Validator{
		validate: validate,
		locale:   locale,
	}

	v.registerCustomValidators() // from rules.go
	return v
}

type ValidationError struct {
	Field   string `json:"field"`
	Message string `json:"message"`
	Tag     string `json:"tag"`
	Value   string `json:"value"`
}

type ValidationErrors []ValidationError

func (ve ValidationErrors) ToKV() map[string]ValidationError {
	m := make(map[string]ValidationError, len(ve))
	for _, err := range ve {
		m[err.Field] = err
	}
	return m
}

func (ve ValidationErrors) Error() string {
	var messages []string
	for _, err := range ve {
		messages = append(messages, fmt.Sprintf("%s: %s", err.Field, err.Message))
	}
	return strings.Join(messages, "; ")
}

func (v *Validator) Validate(i any) error {
	err := v.validate.Struct(i)
	if err == nil {
		return nil
	}

	var validationErrors ValidationErrors
	if validationErrs, ok := err.(validator.ValidationErrors); ok {
		for _, fieldErr := range validationErrs {
			validationError := ValidationError{
				Field:   fieldErr.Field(),
				Tag:     fieldErr.Tag(),
				Value:   fmt.Sprintf("%v", fieldErr.Value()),
				Message: v.getErrorMessage(fieldErr),
			}
			validationErrors = append(validationErrors, validationError)
		}
	}

	return validationErrors
}

func (v *Validator) ValidateAndGetErrors(i any) ValidationErrors {
	err := v.Validate(i)
	if err == nil {
		return nil
	}
	if validationErrs, ok := err.(ValidationErrors); ok {
		return validationErrs
	}
	return nil
}

func (v *Validator) getErrorMessage(fe validator.FieldError) string {
	field := fe.Field()
	param := fe.Param()
	tag := fe.Tag()

	switch strings.ToLower(v.locale) {
	case "id":
		return v.indonesianErrorMessage(field, tag, param)
	case "en":
		return v.englishErrorMessage(field, tag, param)
	default:
		return v.indonesianErrorMessage(field, tag, param)
	}
}

func (v *Validator) registerCustomValidators() {
	// All custom rules are registered here (defined in rules.go)
	v.validate.RegisterValidation("password", v.validatePassword)
	v.validate.RegisterValidation("strong_password", v.validateStrongPassword)
	v.validate.RegisterValidation("phone_id", v.validateIndonesianPhone)
	v.validate.RegisterValidation("postal_code_id", v.validateIndonesianPostalCode)
	v.validate.RegisterValidation("nik", v.validateIndonesianNIK)
	v.validate.RegisterValidation("username", v.validateUsername)
	v.validate.RegisterValidation("no_html", v.validateNoHTML)
	v.validate.RegisterValidation("currency_id", v.validateIndonesianCurrency)
}

// Helpers for safe type checking
func IsValidationErrors(err error) bool {
	_, ok := err.(ValidationErrors)
	return ok
}

func GetValidationErrors(err error) (ValidationErrors, bool) {
	if validationErrs, ok := err.(ValidationErrors); ok {
		return validationErrs, true
	}
	return nil, false
}
```

## 2. rules.go (Custom Rules Technique)

```go
package validator

import (
	"regexp"
	"strings"
	"unicode"

	"github.com/go-playground/validator/v10"
)

// All custom validation functions follow this signature and are methods on *Validator
// (so they can be registered easily)

func (v *Validator) validatePassword(fl validator.FieldLevel) bool { ... }           // see original ethos-go
func (v *Validator) validateStrongPassword(fl validator.FieldLevel) bool { ... }
func (v *Validator) validateIndonesianPhone(fl validator.FieldLevel) bool { ... }
func (v *Validator) validateIndonesianPostalCode(fl validator.FieldLevel) bool { ... }
func (v *Validator) validateIndonesianNIK(fl validator.FieldLevel) bool { ... }
func (v *Validator) validateUsername(fl validator.FieldLevel) bool { ... }
func (v *Validator) validateNoHTML(fl validator.FieldLevel) bool { ... }
func (v *Validator) validateIndonesianCurrency(fl validator.FieldLevel) bool { ... }
```

*(Copy the exact implementations from the ethos-go raw files — they are production-ready and well-commented.)*

## 3. message.go (Localized Messages Technique)

```go
package validator

import "fmt"

func (v *Validator) englishErrorMessage(field, tag, param string) string { ... }
func (v *Validator) indonesianErrorMessage(field, tag, param string) string { ... }
```

*(Again, copy the full switch statements from ethos-go/message.go — they already cover all built-in + custom tags.)*

## Usage in Handlers (with Problem Integration)

```go
// In your main or service setup (once)
var Validator = validator.New("id") // or "en"

type CreateUserRequest struct {
	Name     string `json:"name"     validate:"required,min=3,max=100"`
	Email    string `json:"email"    validate:"required,email"`
	Phone    string `json:"phone"    validate:"required,phone_id"`
	Password string `json:"password" validate:"required,strong_password"`
	// ...
}

func CreateUser(w http.ResponseWriter, r *http.Request) {
	var req CreateUserRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		problem.New(problem.BadRequest, problem.WithDetail("Invalid JSON")).Write(w)
		return
	}

	if validationErrs := Validator.ValidateAndGetErrors(&req); validationErrs != nil {
		p := problem.New(problem.UnprocessableEntity,
			problem.WithDetail("Validation failed"),
			problem.WithInstance(r.URL.Path),
			problem.WithExtension("field_errors", validationErrs.ToKV()), // map for frontend
		)
		p.Write(w)
		return
	}

	// validation passed → business logic
}
```

## Ultra-Clean Helper (Recommended Improvement)

Add this method to `validator.go`:

```go
// ValidateAndWriteProblem returns true if validation failed and wrote the Problem response
func (v *Validator) ValidateAndWriteProblem(w http.ResponseWriter, r *http.Request, i any) bool {
	if errs := v.ValidateAndGetErrors(i); errs != nil {
		problem.New(problem.UnprocessableEntity,
			problem.WithDetail("Validation failed"),
			problem.WithInstance(r.URL.Path),
			problem.WithExtension("field_errors", errs.ToKV()),
		).Write(w)
		return true
	}
	return false
}
```

Usage becomes **one line**:

```go
if Validator.ValidateAndWriteProblem(w, r, &req) {
	return
}
```

## Best Practices & Decision Tree

**Always**:
- Create `var Validator = validator.New("id")` in a shared `internal/common` package.
- Put all request structs in the same package as the handler or in `internal/dto`.
- Use `validate` + `json` tags on every field.
- Always return validation errors via the Problem skill (never manual JSON).

**Decision Tree**:
1. Need a new rule? → Add function to `rules.go` + register in `registerCustomValidators()` + add message in both languages.
2. Support new language? → Add case in `getErrorMessage()` + new method in `message.go`.
3. Want per-request locale? → Add `WithLocale(locale string) *Validator` helper.

**Never**:
- Write manual `if field == ""` checks.
- Duplicate validation logic across files.
- Return 400 for validation errors (use 422).

## Why This Technique Wins
- Tiny, self-contained package (easy to copy between projects)  
- Fully testable  
- Locale switching is trivial  
- Errors are structured and frontend-friendly  
- Extensible without touching handler code  

Copy the three files into every new Go backend, run `go mod tidy`, and you have production-grade validation in minutes.  
When in doubt, follow the exact structure and examples above — this is the official SemmiDev validation design pattern.  

Combine this with the `http-error-handling-with-problem` skill and your backend error handling becomes bulletproof.
