# Package Errors

## Purpose

Custom error types and error handling utilities for the application. Centralizes error definitions to ensure consistent error handling across all layers. **Follows dependency injection pattern** with interface-based error handling.

## Why Custom Errors?

- **Consistency**: Same error types across the application
- **Context**: Add more context to errors
- **Type Checking**: Can check error types programmatically
- **HTTP Mapping**: Easy to map to HTTP status codes
- **Testability**: Can mock error handling in tests

## Error Handler Interface (For Dependency Injection)

While Go's built-in `error` is sufficient for most cases, we provide a custom error handler interface for advanced scenarios:

```go
// error_handler.go
package errors

// ErrorHandler interface for dependency injection (optional)
// Most code will use standard Go errors, but this is useful for
// logging, metrics, or custom error handling behavior
type ErrorHandler interface {
    HandleError(err error, context map[string]interface{}) error
    WrapError(err error, message string) error
    IsRetryable(err error) bool
}
```

## Standard Error Types

### Domain Errors

```go
// errors.go
package errors

import (
    "errors"
    "fmt"
)

// Domain Errors (standard Go errors)
var (
    ErrNotFound         = errors.New("resource not found")
    ErrAlreadyExists    = errors.New("resource already exists")
    ErrInvalidInput     = errors.New("invalid input")
    ErrUnauthorized     = errors.New("unauthorized")
    ErrForbidden        = errors.New("forbidden")
    ErrConflict         = errors.New("resource conflict")
)

// Business Errors
var (
    ErrClientCannotBorrow = errors.New("client cannot borrow more books")
    ErrBookNotAvailable   = errors.New("book is not available")
    ErrBookOverdue        = errors.New("book is overdue")
    ErrInsufficientPoints = errors.New("insufficient points")
)

// External Service Errors
var (
    ErrServiceUnavailable = errors.New("external service unavailable")
    ErrServiceTimeout     = errors.New("service request timeout")
    ErrInvalidResponse    = errors.New("invalid response from service")
)
```

## Custom Error Types

### DomainError (Structured Error with Context)

```go
// domain_error.go
package errors

// DomainError is a custom error with additional context
type DomainError struct {
    Code    string
    Message string
    Err     error
}

func (e *DomainError) Error() string {
    if e.Err != nil {
        return fmt.Sprintf("%s: %v", e.Message, e.Err)
    }
    return e.Message
}

func (e *DomainError) Unwrap() error {
    return e.Err
}

// NewDomainError creates a new domain error
func NewDomainError(code, message string, err error) *DomainError {
    return &DomainError{
        Code:    code,
        Message: message,
        Err:     err,
    }
}

// Common domain error constructors
func NewNotFoundError(resource string) *DomainError {
    return &DomainError{
        Code:    "NOT_FOUND",
        Message: fmt.Sprintf("%s not found", resource),
        Err:     ErrNotFound,
    }
}

func NewValidationError(message string) *DomainError {
    return &DomainError{
        Code:    "VALIDATION_ERROR",
        Message: message,
        Err:     ErrInvalidInput,
    }
}

func NewConflictError(message string) *DomainError {
    return &DomainError{
        Code:    "CONFLICT",
        Message: message,
        Err:     ErrConflict,
    }
}
```

## Error Wrapping

```go
// wrap.go
package errors

import "fmt"

// Wrap wraps an error with additional context
func Wrap(err error, message string) error {
    if err == nil {
        return nil
    }
    return fmt.Errorf("%s: %w", message, err)
}

// Wrapf wraps an error with formatted message
func Wrapf(err error, format string, args ...interface{}) error {
    if err == nil {
        return nil
    }
    message := fmt.Sprintf(format, args...)
    return fmt.Errorf("%s: %w", message, err)
}

// Usage examples:
// return errors.Wrap(err, "failed to get client")
// return errors.Wrapf(err, "failed to get client %s", clientID)
```

## HTTP Error Mapping

```go
// http_errors.go
package errors

import (
    "errors"
    "fmt"
    "net/http"
)

// HTTPError represents an error with HTTP status code
type HTTPError struct {
    StatusCode int
    Message    string
    Err        error
}

func (e *HTTPError) Error() string {
    return e.Message
}

// ToHTTPStatus maps domain errors to HTTP status codes
func ToHTTPStatus(err error) int {
    if err == nil {
        return http.StatusOK
    }
    
    // Check for HTTPError
    var httpErr *HTTPError
    if errors.As(err, &httpErr) {
        return httpErr.StatusCode
    }
    
    // Check for DomainError
    var domainErr *DomainError
    if errors.As(err, &domainErr) {
        switch domainErr.Code {
        case "NOT_FOUND":
            return http.StatusNotFound
        case "VALIDATION_ERROR":
            return http.StatusBadRequest
        case "CONFLICT":
            return http.StatusConflict
        case "UNAUTHORIZED":
            return http.StatusUnauthorized
        case "FORBIDDEN":
            return http.StatusForbidden
        default:
            return http.StatusInternalServerError
        }
    }
    
    // Check for standard errors
    switch {
    case errors.Is(err, ErrNotFound):
        return http.StatusNotFound
    case errors.Is(err, ErrInvalidInput):
        return http.StatusBadRequest
    case errors.Is(err, ErrUnauthorized):
        return http.StatusUnauthorized
    case errors.Is(err, ErrForbidden):
        return http.StatusForbidden
    case errors.Is(err, ErrConflict):
        return http.StatusConflict
    case errors.Is(err, ErrServiceUnavailable):
        return http.StatusServiceUnavailable
    default:
        return http.StatusInternalServerError
    }
}

// NewHTTPError creates a new HTTP error
func NewHTTPError(statusCode int, message string) *HTTPError {
    return &HTTPError{
        StatusCode: statusCode,
        Message:    message,
    }
}

// Common HTTP error constructors
func NewBadRequestError(message string) *HTTPError {
    return NewHTTPError(http.StatusBadRequest, message)
}

func NewNotFoundHTTPError(resource string) *HTTPError {
    return NewHTTPError(http.StatusNotFound, fmt.Sprintf("%s not found", resource))
}

func NewUnauthorizedError() *HTTPError {
    return NewHTTPError(http.StatusUnauthorized, "Unauthorized")
}

func NewForbiddenError() *HTTPError {
    return NewHTTPError(http.StatusForbidden, "Forbidden")
}

func NewInternalError() *HTTPError {
    return NewHTTPError(http.StatusInternalServerError, "Internal server error")
}
```

## Usage in Layers

### Domain Layer

```go
// domain/entities/client.go
package entities

import "ExampleProject/pkg/errors"

func (c *Client) BorrowBook(bookID string) error {
    if !c.CanBorrow() {
        return errors.ErrClientCannotBorrow
    }
    c.BorrowedBooks = append(c.BorrowedBooks, bookID)
    return nil
}
```

### Repository Layer

```go
// infrastructure/repositories/client_repository_impl.go
package repositories

import (
    "database/sql"
    "ExampleProject/pkg/errors"
)

func (r *ClientRepositoryImpl) GetByID(id string) (*entities.Client, error) {
    // ... query database
    if err == sql.ErrNoRows {
        return nil, errors.NewNotFoundError("client")
    }
    return client, nil
}
```

### Use Case Layer

```go
// application/usecases/rent_book.go
package usecases

import "ExampleProject/pkg/errors"

func (uc *RentBookUseCase) Execute(clientID, bookID string) error {
    client, err := uc.clientRepo.GetByID(clientID)
    if err != nil {
        return errors.Wrap(err, "failed to get client")
    }
    
    if !client.CanBorrow() {
        return errors.NewValidationError("client cannot borrow more books")
    }
    
    // ... rest of logic
}
```

### Handler Layer

```go
// interfaces/api/handlers/book_handler.go
package handlers

import "ExampleProject/pkg/errors"

func (h *BookHandler) RentBook(w http.ResponseWriter, r *http.Request) {
    err := h.rentBookUseCase.Execute(req.ClientID, req.BookID)
    if err != nil {
        status := errors.ToHTTPStatus(err)
        h.respondError(w, status, err.Error())
        return
    }
    // ... success response
}
```

## Error Checking

Go provides built-in functions for error checking:

```go
// Check for specific error
if errors.Is(err, ErrNotFound) {
    // Handle not found
}

// Check for error type
var domainErr *DomainError
if errors.As(err, &domainErr) {
    // Access domainErr.Code, domainErr.Message
}

// Unwrap errors
underlying := errors.Unwrap(err)
```

## Advanced: Error Handler Interface (Optional)

For advanced scenarios like logging all errors or collecting metrics:

```go
// error_handler.go
package errors

import "ExampleProject/pkg/logger"

// DefaultErrorHandler provides basic error handling with logging
type DefaultErrorHandler struct {
    logger logger.Logger
}

// NewErrorHandler creates an error handler with dependency injection
func NewErrorHandler(log logger.Logger) ErrorHandler {
    return &DefaultErrorHandler{
        logger: log,
    }
}

func (h *DefaultErrorHandler) HandleError(err error, context map[string]interface{}) error {
    if err == nil {
        return nil
    }
    
    // Log error with context
    fields := make([]logger.Field, 0, len(context)+1)
    fields = append(fields, logger.Error(err))
    
    for key, value := range context {
        fields = append(fields, logger.Any(key, value))
    }
    
    h.logger.Error("Error occurred", fields...)
    
    return err
}

func (h *DefaultErrorHandler) WrapError(err error, message string) error {
    if err == nil {
        return nil
    }
    
    wrapped := Wrap(err, message)
    h.logger.Error("Error wrapped",
        logger.Error(wrapped),
        logger.String("original_error", err.Error()),
    )
    
    return wrapped
}

func (h *DefaultErrorHandler) IsRetryable(err error) bool {
    // Check if error is retryable
    return errors.Is(err, ErrServiceUnavailable) ||
           errors.Is(err, ErrServiceTimeout)
}

// Usage with dependency injection
func main() {
    log := logger.NewSimpleLogger(logger.InfoLevel)
    errorHandler := errors.NewErrorHandler(log)
    
    // Inject into services that need error handling
    service := services.NewService(repo, errorHandler)
}
```

## Testing with Errors

```go
// test example
func TestRentBook_ClientNotFound(t *testing.T) {
    mockRepo := &MockRepository{
        GetByIDFunc: func(id string) (*entities.Client, error) {
            return nil, errors.NewNotFoundError("client")
        },
    }
    
    useCase := usecases.NewRentBookUseCase(mockRepo, nil, nil, nil)
    
    err := useCase.Execute("invalid-id", "book1")
    
    // Check error type
    var domainErr *errors.DomainError
    if !errors.As(err, &domainErr) {
        t.Fatal("Expected DomainError")
    }
    
    if domainErr.Code != "NOT_FOUND" {
        t.Errorf("Expected NOT_FOUND, got %s", domainErr.Code)
    }
}

func TestToHTTPStatus_NotFound(t *testing.T) {
    err := errors.NewNotFoundError("client")
    status := errors.ToHTTPStatus(err)
    
    if status != http.StatusNotFound {
        t.Errorf("Expected 404, got %d", status)
    }
}
```

## Best Practices

1. **Define Once**: Define errors in this package, use everywhere
2. **Wrap Errors**: Add context when returning errors
   ```go
   return errors.Wrap(err, "failed to save client")
   ```
3. **Check Types**: Use `errors.Is()` and `errors.As()`
   ```go
   if errors.Is(err, errors.ErrNotFound) { ... }
   ```
4. **HTTP Mapping**: Map errors to HTTP status codes in handlers
   ```go
   status := errors.ToHTTPStatus(err)
   ```
5. **Log Details**: Log full error details, return sanitized messages to clients
   ```go
   logger.Error("Database error", logger.Error(err))
   return "An error occurred"  // Don't expose internal details
   ```
6. **Don't Expose Internal Details**: Don't leak internal errors to API responses
7. **Use Standard Errors**: Prefer Go's standard `error` for most cases
8. **Custom Errors for Context**: Use `DomainError` when you need structured error info

## Error Flow Example

```
Domain Layer (entities)
    └─> Returns: errors.ErrClientCannotBorrow
           │
           ▼
Repository Layer
    └─> Returns: errors.NewNotFoundError("client")
           │
           ▼
Use Case Layer
    └─> Wraps: errors.Wrap(err, "failed to rent book")
           │
           ▼
Handler Layer
    └─> Maps: errors.ToHTTPStatus(err) → 404
    └─> Returns: JSON with error message
```

## Summary

- ✅ Use standard Go errors for most cases
- ✅ Use `DomainError` for structured errors with codes
- ✅ Use `HTTPError` for HTTP-specific errors
- ✅ Wrap errors to add context
- ✅ Map errors to HTTP status codes in handlers
- ✅ Optional: Use `ErrorHandler` interface for advanced scenarios
- ✅ Always check errors with `errors.Is()` and `errors.As()`
- ✅ Log full details, return sanitized messages

---

**Define your custom errors here for consistent error handling across all layers!**

