# API Handlers

## Purpose

HTTP handlers (controllers) receive HTTP requests, validate them, call use cases, and return HTTP responses. **Handlers follow the interface pattern for dependency injection and testability.**

## Handler Interfaces

Handlers are defined as interfaces to enable:
- Easy mocking in tests
- Flexibility to swap implementations
- Consistent dependency injection pattern

```go
// handler.go
package handlers

import "net/http"

// BookHandler interface for book-related operations
type BookHandler interface {
    RentBook(w http.ResponseWriter, r *http.Request)
    ReturnBook(w http.ResponseWriter, r *http.Request)
}

// ClientHandler interface for client-related operations
type ClientHandler interface {
    GetPromotions(w http.ResponseWriter, r *http.Request)
    GetBooks(w http.ResponseWriter, r *http.Request)
    ConsumePoints(w http.ResponseWriter, r *http.Request)
}
```

## Rules

✅ **DO:**
- Define handler interfaces
- Return interface types from constructors
- Handle HTTP request/response
- Validate input
- Call use cases
- Convert between DTOs and domain entities
- Return appropriate HTTP status codes
- Inject logger for request logging

❌ **DON'T:**
- Include business logic (use use cases)
- Access database directly
- Call external APIs directly
- Skip dependency injection

## Example: Book Handler Implementation

```go
// book_handler.go
package handlers

import (
    "encoding/json"
    "net/http"
    "ExampleProject/application/usecases"
    "ExampleProject/interfaces/dto"
    "ExampleProject/pkg/logger"
)

// BookHandlerImpl implements the BookHandler interface
type BookHandlerImpl struct {
    rentBookUseCase   *usecases.RentBookUseCase
    returnBookUseCase *usecases.ReturnBookUseCase
    logger            logger.Logger
}

// NewBookHandler creates a BookHandler with dependency injection
func NewBookHandler(
    rentBookUseCase *usecases.RentBookUseCase,
    returnBookUseCase *usecases.ReturnBookUseCase,
    log logger.Logger,
) BookHandler {  // Returns interface type
    return &BookHandlerImpl{
        rentBookUseCase:   rentBookUseCase,
        returnBookUseCase: returnBookUseCase,
        logger:            log,
    }
}

func (h *BookHandlerImpl) RentBook(w http.ResponseWriter, r *http.Request) {
    // Create contextual logger
    requestLogger := h.logger.WithFields(
        logger.String("method", r.Method),
        logger.String("path", r.URL.Path),
        logger.String("request_id", r.Header.Get("X-Request-ID")),
    )
    
    requestLogger.Info("Handling rent book request")
    
    // 1. Parse request
    var req dto.RentBookRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        requestLogger.Error("Invalid request body", logger.Error(err))
        h.respondError(w, http.StatusBadRequest, "Invalid request body")
        return
    }
    
    // 2. Validate request
    if err := req.Validate(); err != nil {
        requestLogger.Warn("Validation failed", logger.Error(err))
        h.respondError(w, http.StatusBadRequest, err.Error())
        return
    }
    
    // 3. Execute use case
    err := h.rentBookUseCase.Execute(req.ClientID, req.BookID, req.DurationDays)
    if err != nil {
        requestLogger.Error("Use case failed", logger.Error(err))
        h.respondError(w, http.StatusBadRequest, err.Error())
        return
    }
    
    // 4. Return success response
    requestLogger.Info("Book rented successfully")
    resp := dto.RentBookResponse{
        Success: true,
        Message: "Book rented successfully",
    }
    
    h.respondJSON(w, http.StatusOK, resp)
}

func (h *BookHandlerImpl) ReturnBook(w http.ResponseWriter, r *http.Request) {
    requestLogger := h.logger.WithFields(
        logger.String("method", r.Method),
        logger.String("path", r.URL.Path),
    )
    
    var req dto.ReturnBookRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        h.respondError(w, http.StatusBadRequest, "Invalid request body")
        return
    }
    
    if err := req.Validate(); err != nil {
        h.respondError(w, http.StatusBadRequest, err.Error())
        return
    }
    
    err := h.returnBookUseCase.Execute(req.ClientID, req.BookID)
    if err != nil {
        requestLogger.Error("Failed to return book", logger.Error(err))
        h.respondError(w, http.StatusBadRequest, err.Error())
        return
    }
    
    resp := dto.ReturnBookResponse{
        Success: true,
        Message: "Book returned successfully",
    }
    
    h.respondJSON(w, http.StatusOK, resp)
}

// Helper methods
func (h *BookHandlerImpl) respondJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func (h *BookHandlerImpl) respondError(w http.ResponseWriter, status int, message string) {
    h.respondJSON(w, status, dto.ErrorResponse{
        Error:   true,
        Message: message,
    })
}
```

## Example: Client Handler Implementation

```go
// client_handler.go
package handlers

import (
    "encoding/json"
    "net/http"
    "ExampleProject/application/usecases"
    "ExampleProject/interfaces/dto"
    "ExampleProject/pkg/logger"
)

// ClientHandlerImpl implements the ClientHandler interface
type ClientHandlerImpl struct {
    checkPromotionsUseCase *usecases.CheckPromotionsUseCase
    getClientBooksUseCase  *usecases.GetClientBooksUseCase
    consumePointsUseCase   *usecases.ConsumePointsUseCase
    logger                 logger.Logger
}

// NewClientHandler creates a ClientHandler with dependency injection
func NewClientHandler(
    checkPromotionsUseCase *usecases.CheckPromotionsUseCase,
    getClientBooksUseCase *usecases.GetClientBooksUseCase,
    consumePointsUseCase *usecases.ConsumePointsUseCase,
    log logger.Logger,
) ClientHandler {  // Returns interface type
    return &ClientHandlerImpl{
        checkPromotionsUseCase: checkPromotionsUseCase,
        getClientBooksUseCase:  getClientBooksUseCase,
        consumePointsUseCase:   consumePointsUseCase,
        logger:                 log,
    }
}

func (h *ClientHandlerImpl) GetPromotions(w http.ResponseWriter, r *http.Request) {
    clientID := r.URL.Query().Get("client_id")
    if clientID == "" {
        h.respondError(w, http.StatusBadRequest, "client_id is required")
        return
    }
    
    h.logger.Info("Getting promotions", logger.String("client_id", clientID))
    
    result, err := h.checkPromotionsUseCase.Execute(clientID)
    if err != nil {
        h.logger.Error("Failed to get promotions", logger.Error(err))
        h.respondError(w, http.StatusInternalServerError, err.Error())
        return
    }
    
    resp := dto.ToPromotionsResponse(result)
    h.respondJSON(w, http.StatusOK, resp)
}

func (h *ClientHandlerImpl) GetBooks(w http.ResponseWriter, r *http.Request) {
    clientID := r.URL.Query().Get("client_id")
    if clientID == "" {
        h.respondError(w, http.StatusBadRequest, "client_id is required")
        return
    }
    
    books, err := h.getClientBooksUseCase.Execute(clientID)
    if err != nil {
        h.respondError(w, http.StatusInternalServerError, err.Error())
        return
    }
    
    resp := dto.ToBooksResponse(books)
    h.respondJSON(w, http.StatusOK, resp)
}

func (h *ClientHandlerImpl) ConsumePoints(w http.ResponseWriter, r *http.Request) {
    var req dto.ConsumePointsRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        h.respondError(w, http.StatusBadRequest, "Invalid request body")
        return
    }
    
    if err := req.Validate(); err != nil {
        h.respondError(w, http.StatusBadRequest, err.Error())
        return
    }
    
    err := h.consumePointsUseCase.Execute(req.ClientID, req.Points)
    if err != nil {
        h.respondError(w, http.StatusBadRequest, err.Error())
        return
    }
    
    resp := dto.ConsumePointsResponse{
        Success: true,
        Message: "Points consumed successfully",
    }
    
    h.respondJSON(w, http.StatusOK, resp)
}

// Helper methods
func (h *ClientHandlerImpl) respondJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func (h *ClientHandlerImpl) respondError(w http.ResponseWriter, status int, message string) {
    h.respondJSON(w, status, dto.ErrorResponse{
        Error:   true,
        Message: message,
    })
}
```

## HTTP Status Codes Guide

### Success (2xx)
- `200 OK` - Request succeeded
- `201 Created` - Resource created
- `204 No Content` - Success with no response body

### Client Errors (4xx)
- `400 Bad Request` - Invalid input
- `401 Unauthorized` - Authentication required
- `403 Forbidden` - No permission
- `404 Not Found` - Resource not found
- `409 Conflict` - Resource conflict

### Server Errors (5xx)
- `500 Internal Server Error` - Unexpected error
- `503 Service Unavailable` - Service down

## Middleware Example

```go
// middleware.go
package handlers

import (
    "log"
    "net/http"
    "time"
)

// LoggingMiddleware logs all HTTP requests
func LoggingMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        log.Printf("[%s] %s %s", r.Method, r.URL.Path, r.RemoteAddr)
        
        next(w, r)
        
        log.Printf("[%s] %s completed in %v", r.Method, r.URL.Path, time.Since(start))
    }
}

// CORSMiddleware adds CORS headers
func CORSMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
        
        if r.Method == "OPTIONS" {
            w.WriteHeader(http.StatusOK)
            return
        }
        
        next(w, r)
    }
}

// AuthMiddleware checks authentication
func AuthMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        
        if token == "" {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
        
        // Validate token here
        // ...
        
        next(w, r)
    }
}
```

## Router Setup

```go
// router.go (in interfaces/api/)
package api

import (
    "net/http"
    "ExampleProject/interfaces/api/handlers"
)

func SetupRoutes(
    bookHandler handlers.BookHandler,      // Interface type
    clientHandler handlers.ClientHandler,  // Interface type
) *http.ServeMux {
    mux := http.NewServeMux()
    
    // Book routes
    mux.HandleFunc("/api/books/rent", bookHandler.RentBook)
    mux.HandleFunc("/api/books/return", bookHandler.ReturnBook)
    
    // Client routes
    mux.HandleFunc("/api/clients/promotions", clientHandler.GetPromotions)
    mux.HandleFunc("/api/clients/books", clientHandler.GetBooks)
    mux.HandleFunc("/api/clients/points/consume", clientHandler.ConsumePoints)
    
    // Health check
    mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        w.Write([]byte("OK"))
    })
    
    return mux
}
```

## Testing Handlers

### Unit Testing with Mock Use Cases

```go
// book_handler_test.go
package handlers_test

import (
    "bytes"
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "testing"
    "ExampleProject/interfaces/api/handlers"
    "ExampleProject/interfaces/dto"
    "ExampleProject/pkg/logger"
)

// Mock use case
type MockRentBookUseCase struct {
    ExecuteFunc func(clientID, bookID string, days int) error
}

func (m *MockRentBookUseCase) Execute(clientID, bookID string, days int) error {
    if m.ExecuteFunc != nil {
        return m.ExecuteFunc(clientID, bookID, days)
    }
    return nil
}

func TestBookHandler_RentBook_Success(t *testing.T) {
    // Arrange
    mockUseCase := &MockRentBookUseCase{
        ExecuteFunc: func(clientID, bookID string, days int) error {
            return nil
        },
    }
    
    mockLogger := logger.NewNoOpLogger()
    handler := handlers.NewBookHandler(mockUseCase, nil, mockLogger)
    
    reqBody := dto.RentBookRequest{
        ClientID:     "client1",
        BookID:       "book1",
        DurationDays: 14,
    }
    body, _ := json.Marshal(reqBody)
    
    req := httptest.NewRequest(http.MethodPost, "/api/books/rent", bytes.NewBuffer(body))
    w := httptest.NewRecorder()
    
    // Act
    handler.RentBook(w, req)
    
    // Assert
    if w.Code != http.StatusOK {
        t.Errorf("Expected status 200, got %d", w.Code)
    }
    
    var resp dto.RentBookResponse
    json.NewDecoder(w.Body).Decode(&resp)
    
    if !resp.Success {
        t.Error("Expected success to be true")
    }
}
```

### Integration Testing with Mock Handlers

```go
// router_test.go
package api_test

import (
    "net/http"
    "net/http/httptest"
    "testing"
    "ExampleProject/interfaces/api"
    "ExampleProject/interfaces/api/handlers"
)

// Mock handler
type MockBookHandler struct{}

func (m *MockBookHandler) RentBook(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    w.Write([]byte(`{"success":true}`))
}

func (m *MockBookHandler) ReturnBook(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
}

func TestRouter_BookRoutes(t *testing.T) {
    // Use mock handlers
    mockBookHandler := &MockBookHandler{}
    mockClientHandler := &MockClientHandler{}
    
    router := api.SetupRoutes(mockBookHandler, mockClientHandler)
    
    req := httptest.NewRequest(http.MethodPost, "/api/books/rent", nil)
    w := httptest.NewRecorder()
    
    router.ServeHTTP(w, req)
    
    if w.Code != http.StatusOK {
        t.Errorf("Expected 200, got %d", w.Code)
    }
}
```

## Dependency Injection Flow

```
main.go
  │
  ├─> Creates logger
  ├─> Creates use cases
  │
  └─> Creates handlers (inject use cases & logger)
      │
      └─> handlers.NewBookHandler(
            rentUseCase,
            returnUseCase,
            logger,
          ) → returns handlers.BookHandler interface
```

## Best Practices

1. **Interfaces**: Define and return handler interfaces
2. **Separation**: Keep handlers thin - business logic in use cases
3. **Validation**: Validate input in handlers or DTOs
4. **Error Handling**: Return appropriate HTTP status codes
5. **Logging**: Log all requests and errors with contextual information
6. **Middleware**: Use middleware for cross-cutting concerns
7. **Testing**: Test handlers with mocks and httptest package
8. **Dependency Injection**: Always inject dependencies via constructors

---

**Place your HTTP handlers here following the interface pattern!**

