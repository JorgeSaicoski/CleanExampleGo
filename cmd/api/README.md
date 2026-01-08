# API Application Entry Point

## Purpose

This is the **main entry point** for the API application. This is where you:
- Initialize all dependencies
- Wire everything together (dependency injection)
- Start the HTTP server
- Handle graceful shutdown

## Example: Main Application with Full Dependency Injection

```go
// main.go
package main

import (
    "context"
    "database/sql"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
    
    "ExampleProject/application/usecases"
    "ExampleProject/infrastructure/postgresql"
    pgRepos "ExampleProject/infrastructure/postgresql/repositories"
    "ExampleProject/infrastructure/loyalty_points"
    "ExampleProject/infrastructure/promotion"
    "ExampleProject/infrastructure/notification"
    "ExampleProject/interfaces/api"
    "ExampleProject/interfaces/api/handlers"
    "ExampleProject/pkg/logger"
)

func main() {
    // Initialize logger
    log := logger.NewSimpleLogger(logger.InfoLevel)
    log.Info("Starting application")
    
    // Load configuration
    config := loadConfig()
    
    // Initialize database connection (DI: inject config)
    db, err := postgresql.NewConnection(postgresql.Config{
        Host:     config.DBHost,
        Port:     config.DBPort,
        User:     config.DBUser,
        Password: config.DBPassword,
        Database: config.DBName,
        SSLMode:  config.DBSSLMode,
    })
    if err != nil {
        log.Fatal("Failed to connect to database", logger.Error(err))
    }
    defer db.Close()
    log.Info("Database connected")
    
    // Initialize repositories (DI: inject *sql.DB)
    clientRepo := pgRepos.NewPostgreSQLClientRepository(db)
    bookRepo := pgRepos.NewPostgreSQLBookRepository(db)
    
    // Initialize external services (DI: inject configuration)
    promotionSvc := promotion.NewPromotionService(
        config.PromotionServiceURL,
        config.PromotionAPIKey,
    )
    
    pointsSvc := loyalty_points.NewLoyaltyPointsService(
        config.LoyaltyServiceURL,
        config.LoyaltyAPIKey,
    )
    
    notificationSvc := notification.NewEmailService(
        config.SMTPHost,
        config.SMTPPort,
        config.SMTPUser,
        config.SMTPPassword,
        config.FromEmail,
    )
    
    // Initialize use cases (DI: inject repositories and services)
    rentBookUseCase := usecases.NewRentBookUseCase(
        clientRepo,
        bookRepo,
        promotionSvc,
        pointsSvc,
    )
    
    returnBookUseCase := usecases.NewReturnBookUseCase(
        clientRepo,
        bookRepo,
        pointsSvc,
        notificationSvc,
    )
    
    checkPromotionsUseCase := usecases.NewCheckPromotionsUseCase(
        clientRepo,
        promotionSvc,
    )
    
    getClientBooksUseCase := usecases.NewGetClientBooksUseCase(
        clientRepo,
        bookRepo,
    )
    
    consumePointsUseCase := usecases.NewConsumePointsUseCase(
        clientRepo,
        pointsSvc,
    )
    
    // Initialize handlers (DI: inject use cases)
    bookHandler := handlers.NewBookHandler(
        rentBookUseCase,
        returnBookUseCase,
    )
    
    clientHandler := handlers.NewClientHandler(
        checkPromotionsUseCase,
        getClientBooksUseCase,
        consumePointsUseCase,
    )
    
    // Setup routes
    router := api.SetupRoutes(bookHandler, clientHandler)
    
    // Create server
    server := &http.Server{
        Addr:         ":" + config.Port,
        Handler:      router,
        ReadTimeout:  15 * time.Second,
        WriteTimeout: 15 * time.Second,
        IdleTimeout:  60 * time.Second,
    }
    
    // Start server in goroutine
    go func() {
        log.Info("Server starting", logger.String("port", config.Port))
        if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatal("Server failed to start", logger.Error(err))
        }
    }()
    
    // Wait for interrupt signal for graceful shutdown
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    
    log.Info("Shutting down server...")
    
    // Graceful shutdown with timeout
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    
    if err := server.Shutdown(ctx); err != nil {
        log.Fatal("Server forced to shutdown", logger.Error(err))
    }
    
    log.Info("Server exited")
}

func initDatabase(url string) (*sql.DB, error) {
    db, err := sql.Open("mysql", url)
    if err != nil {
        return nil, err
    }
    
    // Configure connection pool
    db.SetMaxOpenConns(25)
    db.SetMaxIdleConns(5)
    db.SetConnMaxLifetime(5 * time.Minute)
    
    // Test connection
    if err := db.Ping(); err != nil {
        return nil, err
    }
    
    return db, nil
}

type Config struct {
    Port                string
    DatabaseURL         string
    PromotionServiceURL string
    PointsServiceURL    string
    PointsAPIKey        string
    SMTPHost            string
    SMTPPort            int
    SMTPUser            string
    SMTPPassword        string
    FromEmail           string
}

func loadConfig() *Config {
    return &Config{
        Port:                getEnv("PORT", "8080"),
        DatabaseURL:         getEnv("DATABASE_URL", "root:password@tcp(localhost:3306)/library"),
        PromotionServiceURL: getEnv("PROMOTION_SERVICE_URL", "http://localhost:8081"),
        PointsServiceURL:    getEnv("POINTS_SERVICE_URL", "http://localhost:8082"),
        PointsAPIKey:        getEnv("POINTS_API_KEY", ""),
        SMTPHost:            getEnv("SMTP_HOST", "localhost"),
        SMTPPort:            getEnvAsInt("SMTP_PORT", 587),
        SMTPUser:            getEnv("SMTP_USER", ""),
        SMTPPassword:        getEnv("SMTP_PASSWORD", ""),
        FromEmail:           getEnv("FROM_EMAIL", "noreply@library.com"),
    }
}

func getEnv(key, defaultValue string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }
    return defaultValue
}

func getEnvAsInt(key string, defaultValue int) int {
    if value := os.Getenv(key); value != "" {
        if intValue, err := strconv.Atoi(value); err == nil {
            return intValue
        }
    }
    return defaultValue
}
```

## Configuration Options

### Environment Variables
```bash
# Server
PORT=8080

# Database (PostgreSQL)
DB_HOST=localhost
DB_PORT=5432
DB_USER=library_user
DB_PASSWORD=secret
DB_NAME=library_db
DB_SSLMODE=disable

# External Services
PROMOTION_SERVICE_URL=http://promotion-service:8081
PROMOTION_API_KEY=your-promotion-api-key

LOYALTY_SERVICE_URL=http://loyalty-service:8082
LOYALTY_API_KEY=your-loyalty-api-key

# Email
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASSWORD=your-password
FROM_EMAIL=noreply@library.com
```

### Using .env File
```go
import "github.com/joho/godotenv"

func loadConfig() *Config {
    // Load .env file
    godotenv.Load()
    
    return &Config{
        Port: getEnv("PORT", "8080"),
        // ...
    }
}
```

## Graceful Shutdown

The example includes graceful shutdown which:
1. Waits for interrupt signal (Ctrl+C)
2. Stops accepting new requests
3. Waits for existing requests to complete (with timeout)
4. Closes database connections
5. Exits cleanly

## Health Checks

Add health check endpoint:

```go
func setupHealthCheck(db *sql.DB) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        // Check database
        if err := db.Ping(); err != nil {
            w.WriteHeader(http.StatusServiceUnavailable)
            json.NewEncoder(w).Encode(map[string]string{
                "status": "unhealthy",
                "error":  err.Error(),
            })
            return
        }
        
        w.WriteHeader(http.StatusOK)
        json.NewEncoder(w).Encode(map[string]string{
            "status": "healthy",
        })
    }
}

// In main()
http.HandleFunc("/health", setupHealthCheck(db))
```

## Running the Application

```bash
# Development
go run main.go

# Build
go build -o api main.go

# Run with environment variables
PORT=3000 DATABASE_URL=... ./api

# Run with .env file
./api
```

## Docker Support

```dockerfile
# Dockerfile
FROM golang:1.21-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o api cmd/api/main.go

FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /root/

COPY --from=builder /app/api .

EXPOSE 8080
CMD ["./api"]
```

```bash
# Build and run
docker build -t library-api .
docker run -p 8080:8080 \
    -e DATABASE_URL=... \
    -e PROMOTION_SERVICE_URL=... \
    library-api
```

## Best Practices

1. **Dependency Injection**: Wire all dependencies in main
2. **Configuration**: Use environment variables
3. **Graceful Shutdown**: Always implement graceful shutdown
4. **Health Checks**: Add health check endpoints
5. **Logging**: Log startup, shutdown, and errors
6. **Connection Pooling**: Configure database connection pool
7. **Timeouts**: Set appropriate server timeouts
8. **Error Handling**: Handle startup errors gracefully

---

**This is where your application comes to life!**

