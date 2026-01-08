# Package Logger

## Purpose

Centralized logging utilities for the application. Provides structured logging with consistent format across all layers.

## Why Structured Logging?

- **Searchable**: Easy to search logs in log aggregation tools
- **Parseable**: Logs can be parsed and analyzed
- **Consistent**: Same format everywhere
- **Contextual**: Include context like request ID, user ID, etc.

## Example: Logger Interface

```go
// logger.go
package logger

import (
    "fmt"
    "log"
    "os"
    "time"
)

// Logger interface for structured logging
type Logger interface {
    Debug(message string, fields ...Field)
    Info(message string, fields ...Field)
    Warn(message string, fields ...Field)
    Error(message string, fields ...Field)
    Fatal(message string, fields ...Field)
}

// Field represents a log field (key-value pair)
type Field struct {
    Key   string
    Value interface{}
}

// Helper functions to create fields
func String(key, value string) Field {
    return Field{Key: key, Value: value}
}

func Int(key string, value int) Field {
    return Field{Key: key, Value: value}
}

func Error(err error) Field {
    return Field{Key: "error", Value: err.Error()}
}

func Duration(key string, value time.Duration) Field {
    return Field{Key: key, Value: value.String()}
}

func Any(key string, value interface{}) Field {
    return Field{Key: key, Value: value}
}
```

## Example: Simple Logger Implementation

```go
// simple_logger.go
package logger

import (
    "encoding/json"
    "fmt"
    "log"
    "os"
    "time"
)

type SimpleLogger struct {
    logger *log.Logger
    level  LogLevel
}

type LogLevel int

const (
    DebugLevel LogLevel = iota
    InfoLevel
    WarnLevel
    ErrorLevel
    FatalLevel
)

func NewSimpleLogger(level LogLevel) *SimpleLogger {
    return &SimpleLogger{
        logger: log.New(os.Stdout, "", 0),
        level:  level,
    }
}

func (l *SimpleLogger) Debug(message string, fields ...Field) {
    if l.level <= DebugLevel {
        l.log("DEBUG", message, fields...)
    }
}

func (l *SimpleLogger) Info(message string, fields ...Field) {
    if l.level <= InfoLevel {
        l.log("INFO", message, fields...)
    }
}

func (l *SimpleLogger) Warn(message string, fields ...Field) {
    if l.level <= WarnLevel {
        l.log("WARN", message, fields...)
    }
}

func (l *SimpleLogger) Error(message string, fields ...Field) {
    if l.level <= ErrorLevel {
        l.log("ERROR", message, fields...)
    }
}

func (l *SimpleLogger) Fatal(message string, fields ...Field) {
    l.log("FATAL", message, fields...)
    os.Exit(1)
}

func (l *SimpleLogger) log(level, message string, fields ...Field) {
    entry := map[string]interface{}{
        "timestamp": time.Now().Format(time.RFC3339),
        "level":     level,
        "message":   message,
    }
    
    for _, field := range fields {
        entry[field.Key] = field.Value
    }
    
    jsonEntry, _ := json.Marshal(entry)
    l.logger.Println(string(jsonEntry))
}
```

## Example: Contextual Logger

```go
// context_logger.go
package logger

type ContextLogger struct {
    base   Logger
    fields []Field
}

// WithFields returns a new logger with additional fields
func WithFields(base Logger, fields ...Field) Logger {
    return &ContextLogger{
        base:   base,
        fields: fields,
    }
}

func (l *ContextLogger) Debug(message string, fields ...Field) {
    l.base.Debug(message, append(l.fields, fields...)...)
}

func (l *ContextLogger) Info(message string, fields ...Field) {
    l.base.Info(message, append(l.fields, fields...)...)
}

func (l *ContextLogger) Warn(message string, fields ...Field) {
    l.base.Warn(message, append(l.fields, fields...)...)
}

func (l *ContextLogger) Error(message string, fields ...Field) {
    l.base.Error(message, append(l.fields, fields...)...)
}

func (l *ContextLogger) Fatal(message string, fields ...Field) {
    l.base.Fatal(message, append(l.fields, fields...)...)
}

// Usage:
// requestLogger := logger.WithFields(baseLogger,
//     logger.String("request_id", requestID),
//     logger.String("user_id", userID),
// )
// requestLogger.Info("Processing request")
```

## Usage Examples

### Basic Usage
```go
package main

import "ExampleProject/pkg/logger"

func main() {
    log := logger.NewSimpleLogger(logger.InfoLevel)
    
    log.Info("Application started",
        logger.String("version", "1.0.0"),
        logger.Int("port", 8080),
    )
    
    log.Error("Failed to connect to database",
        logger.String("host", "localhost"),
        logger.Int("port", 5432),
        logger.Error(err),
    )
}
```

### In Use Cases
```go
// application/usecases/rent_book.go
type RentBookUseCase struct {
    clientRepo repositories.ClientRepository
    logger     logger.Logger
}

func (uc *RentBookUseCase) Execute(clientID, bookID string) error {
    uc.logger.Info("Renting book",
        logger.String("client_id", clientID),
        logger.String("book_id", bookID),
    )
    
    client, err := uc.clientRepo.GetByID(clientID)
    if err != nil {
        uc.logger.Error("Failed to get client",
            logger.String("client_id", clientID),
            logger.Error(err),
        )
        return err
    }
    
    uc.logger.Debug("Client loaded",
        logger.String("client_name", client.Name),
        logger.Int("borrowed_books", len(client.BorrowedBooks)),
    )
    
    // ... rest of logic
    
    uc.logger.Info("Book rented successfully",
        logger.String("client_id", clientID),
        logger.String("book_id", bookID),
    )
    
    return nil
}
```

### In Handlers
```go
// interfaces/api/handlers/book_handler.go
func (h *BookHandler) RentBook(w http.ResponseWriter, r *http.Request) {
    requestID := r.Header.Get("X-Request-ID")
    
    requestLogger := logger.WithFields(h.logger,
        logger.String("request_id", requestID),
        logger.String("method", r.Method),
        logger.String("path", r.URL.Path),
    )
    
    requestLogger.Info("Handling rent book request")
    
    // ... handle request
    
    requestLogger.Info("Request completed",
        logger.Int("status", http.StatusOK),
    )
}
```

### Timing Operations
```go
func (uc *RentBookUseCase) Execute(clientID, bookID string) error {
    start := time.Now()
    defer func() {
        uc.logger.Info("Use case completed",
            logger.Duration("duration", time.Since(start)),
        )
    }()
    
    // ... execute use case
}
```

## Integration with Popular Loggers

### Using Zap
```go
// zap_logger.go
package logger

import "go.uber.org/zap"

type ZapLogger struct {
    logger *zap.Logger
}

func NewZapLogger() (*ZapLogger, error) {
    zapLogger, err := zap.NewProduction()
    if err != nil {
        return nil, err
    }
    
    return &ZapLogger{logger: zapLogger}, nil
}

func (l *ZapLogger) Info(message string, fields ...Field) {
    zapFields := make([]zap.Field, len(fields))
    for i, f := range fields {
        zapFields[i] = zap.Any(f.Key, f.Value)
    }
    l.logger.Info(message, zapFields...)
}

// Implement other methods...
```

### Using Logrus
```go
// logrus_logger.go
package logger

import "github.com/sirupsen/logrus"

type LogrusLogger struct {
    logger *logrus.Logger
}

func NewLogrusLogger() *LogrusLogger {
    logger := logrus.New()
    logger.SetFormatter(&logrus.JSONFormatter{})
    
    return &LogrusLogger{logger: logger}
}

func (l *LogrusLogger) Info(message string, fields ...Field) {
    logFields := logrus.Fields{}
    for _, f := range fields {
        logFields[f.Key] = f.Value
    }
    l.logger.WithFields(logFields).Info(message)
}

// Implement other methods...
```

## Log Output Examples

### JSON Format (Recommended for Production)
```json
{"timestamp":"2024-01-02T15:04:05Z","level":"INFO","message":"Book rented","client_id":"123","book_id":"456"}
{"timestamp":"2024-01-02T15:04:06Z","level":"ERROR","message":"Failed to connect","error":"connection refused"}
```

### Human-Readable Format (Good for Development)
```
2024-01-02 15:04:05 INFO  Book rented client_id=123 book_id=456
2024-01-02 15:04:06 ERROR Failed to connect error="connection refused"
```

## Best Practices

1. **Use Structured Logging**: Always use key-value pairs
2. **Consistent Keys**: Use same keys across application (e.g., "client_id" not "clientID")
3. **Log Levels**: Use appropriate log levels
   - DEBUG: Detailed information for diagnosing problems
   - INFO: General informational messages
   - WARN: Warning messages for potentially harmful situations
   - ERROR: Error messages for failures
   - FATAL: Critical errors that cause shutdown
4. **Don't Log Sensitive Data**: Never log passwords, tokens, credit cards
5. **Add Context**: Include relevant context (IDs, timestamps, durations)
6. **Performance**: Avoid expensive operations in log statements
7. **Request IDs**: Include request ID for tracing requests

## Summary: Logger Dependency Injection

```
main.go
  │
  ├─> Creates logger: logger.NewSimpleLogger(InfoLevel)
  │
  ├─> Injects into Repositories
  │   └─> repositories.NewPostgreSQLClientRepository(db, logger)
  │
  ├─> Injects into Services
  │   └─> services.NewPromotionService(url, key, logger)
  │
  ├─> Injects into Use Cases
  │   └─> usecases.NewRentBookUseCase(..., logger)
  │
  └─> Injects into Handlers
      └─> handlers.NewBookHandler(useCase, logger)
```

**Key Points:**
- ✅ Logger is an interface (easy to mock)
- ✅ Logger is injected via constructors (DI pattern)
- ✅ No global logger variables
- ✅ Easy to swap implementations (JSON, text, third-party)
- ✅ Testable with mock loggers
- ✅ Contextual logging with WithFields()

---

**Use this package for all logging needs - always inject the logger interface!**

