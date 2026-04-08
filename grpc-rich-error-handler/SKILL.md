---
name: grpc-rich-error-handler
description: Implements advanced gRPC error handling in Go using the Rich Error Model with structured context, sanitization, and helper functions for clean code.
---

## 1. Struktur Project & Import
Tetap gunakan import standar ini. Pastikan versi `genproto` sudah up-to-date.

```go
import (
	"context"
	"fmt"
	"log"

	"google.golang.org/genproto/googleapis/rpc/errdetails"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
)
```

## 2. Sisi Server: Menggunakan Helper Functions (Clean Code)
Alih-alih menulis logika `status` di dalam fungsi bisnis, buatlah fungsi pembantu (helper) di file terpisah (misalnya `errors.go`).

```go
package main

import (
	"context"
	"fmt"
	"log"

	"google.golang.org/genproto/googleapis/rpc/errdetails"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
)

// --- HELPER FUNCTIONS (Error Factory) ---

// NewValidationError helper untuk error validasi input (400 Bad Request)
func NewValidationError(msg string, violations []*errdetails.BadRequest_FieldViolation) error {
	st := status.New(codes.InvalidArgument, msg)
	details := &errdetails.BadRequest{FieldViolations: violations}
	
	// Attach detail, jika gagal fallback ke error biasa
	if stWithDetails, err := st.WithDetails(details); err == nil {
		return stWithDetails.Err()
	}
	return st.Err()
}

// NewInternalError helper untuk error sistem (500 Internal Server Error)
// Menerima 'err' asli untuk logging server, dan 'userMsg' untuk response klien.
func NewInternalError(ctx context.Context, err error, userMsg string) error {
	// 1. Generate Tracking ID (sebaiknya menggunakan UUID lib)
	trackingID := fmt.Sprintf("TRK-%d", 12345) 

	// 2. Log error asli (Internal Only)
	log.Printf("[ERROR] Context: %v | TrackingID: %s | Error: %v", ctx, trackingID, err)

	// 3. Buat status aman untuk klien
	st := status.New(codes.Internal, userMsg)
	
	// 4. Tambahkan ErrorInfo (Metadata tracking ID)
	info := &errdetails.ErrorInfo{
		Reason: "INTERNAL_ERROR",
		Domain: "user.service.com",
		Metadata: map[string]string{
			"tracking_id": trackingID,
		},
	}

	if stWithDetails, err := st.WithDetails(info); err == nil {
		return stWithDetails.Err()
	}
	return st.Err()
}

// --- IMPLEMENTASI SERVER ---

type MyServiceServer struct{}

func (s *MyServiceServer) CreateUser(ctx context.Context, req *CreateUserRequest) (*CreateUserResponse, error) {
	
	// CASE 1: Validation Error (menggunakan Helper)
	if req.Email == "" {
		violations := []*errdetails.BadRequest_FieldViolation{
			{
				Field:       "email",
				Description: "Email tidak boleh kosong",
			},
		}
		return nil, NewValidationError("Data registrasi tidak valid", violations)
	}

	// CASE 2: Internal Error (menggunakan Helper)
	// Misal ini error dari database
	errDB := saveToDatabase(req)
	if errDB != nil {
		// Kita pass error DB asli ke helper agar di-log, 
		// tapi klien hanya dapat pesan "Terjadi kesalahan..."
		return nil, NewInternalError(ctx, errDB, "Terjadi kesalahan pada sistem kami. Silakan coba lagi nanti.")
	}

	return &CreateUserResponse{Id: "123", Message: "Success"}, nil
}

func saveToDatabase(req any) error {
	return fmt.Errorf("connection refused: db timeout")
}
```

## 3. Sisi Klien: Error Interceptor/Parser
Di sisi klien, kita perlu mengekstrak informasi tersebut dengan robust. Pendekatan terbaik adalah membuat fungsi yang mengembalikan struct yang rapi, bukan membiarkan logic parsing bercampur dengan business logic.

```go
package main

import (
	"context"
	"fmt"

	"google.golang.org/genproto/googleapis/rpc/errdetails"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
)

// ErrorDetail struct untuk menyimpan hasil parsing error gRPC
type GRPCError struct {
	Code      codes.Code
	Message   string
	IsSystem  bool   // true jika 500/Internal, false jika 400/Validation
	Fields    map[string]string // Untuk validasi field
	TrackingID string           // Untuk internal error tracking
}

// ParseError mengubah error gRPC mentah menjadi struct yang mudah digunakan
func ParseError(err error) *GRPCError {
	st, ok := status.FromError(err)
	if !ok {
		// Error bukan gRPC (misal network error)
		return &GRPCError{
			Code:    codes.Unknown,
			Message: err.Error(),
			IsSystem: true,
		}
	}

	result := &GRPCError{
		Code:    st.Code(),
		Message: st.Message(),
		Fields:  make(map[string]string),
	}

	// Cek apakah ini Internal Server Error
	if st.Code() == codes.Internal {
		result.IsSystem = true
	}

	// Iterate details untuk mengekstrak metadata
	for _, detail := range st.Details() {
		switch d := detail.(type) {
		case *errdetails.BadRequest:
			// Mapping field violations
			for _, v := range d.GetFieldViolations() {
				result.Fields[v.GetField()] = v.GetDescription()
			}
		case *errdetails.ErrorInfo:
			// Ambil tracking ID dari metadata
			if tid, exists := d.GetMetadata()["tracking_id"]; exists {
				result.TrackingID = tid
			}
		}
	}

	return result
}

// --- CONTOH PENGGUNAAN DI KLIEN ---

func callCreateUser(client MyServiceClient) {
	req := &CreateUserRequest{Email: ""} // Trigger error
	
	resp, err := client.CreateUser(context.Background(), req)
	if err != nil {
		parsedErr := ParseError(err)

		fmt.Printf("Terjadi Error: [%s] %s\n", parsedErr.Code, parsedErr.Message)

		if parsedErr.IsSystem {
			fmt.Printf(">> Mohon hubungi CS dengan Tracking ID: %s\n", parsedErr.TrackingID)
			return
		}

		// Handle Validasi Field
		if len(parsedErr.Fields) > 0 {
			fmt.Println(">> Validasi Gagal:")
			for field, msg := range parsedErr.Fields {
				fmt.Printf("   - %s: %s\n", field, msg)
			}
		}
		return
	}

	fmt.Printf("Sukses: %v\n", resp)
}
```

## Ringkasan Improvements

1.  **Helper Functions (`NewValidationError`, `NewInternalError`):** Ini adalah kunci utama *clean code*. Logic bisnis (`CreateUser`) tidak lagi tercemari oleh kode boilerplate gRPC.
2.  **Parsable Struct di Klien:** Dengan membuat fungsi `ParseError`, kode pemanggil (consumer) tidak perlu tahu detail implementasi protobuf (`switch t := detail.(type)`). Cukup akses `.Fields` atau `.TrackingID`.
3.  **Security:** Error asli (seperti SQL error) **selalu** di-log di server dan **tidak pernah** dikirim ke klien. Klien hanya menerima Tracking ID untuk pelacakan.

## Kesimpulan
Kode awal Anda sudah benar secara logika. Versi di atas meningkatkannya dari segi **maintainability** dan **usability** dengan memperkenalkan layer pembantu (abstraction) untuk pembuatan dan pembacaan error.
