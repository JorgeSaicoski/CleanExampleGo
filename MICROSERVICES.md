# Go Satellite Microservice Blueprint

This document defines the architectural standard for **satellite microservices** running alongside the Clean Architecture monolith.

Unlike the monolith's 4-layer structure, satellite microservices are built using a **pragmatic, package-by-feature architecture** with a **centralized HTTP router** and a **dual-ingress model** (HTTP APIs + Asynchronous Event Consumer).

---

## 📁 Directory Structure

```
MicroserviceExampleGo/
├── cmd/
│   └── api/
│       └── main.go                  # Composition root & graceful shutdown
│
├── internal/
│   ├── transport/                   # Inbound driving adapters
│   │   ├── http/
│   │   │   ├── router.go            # Centralized router (routes & middleware registration)
│   │   │   ├── middleware.go        # Logging, recovery, auth headers
│   │   │   └── response.go          # Standard JSON helpers (Respond, RespondError)
│   │   │
│   │   └── event/
│   │       └── consumer.go          # Asynchronous event worker (RabbitMQ/SQS/Kafka)
│   │
│   ├── review/                      # Feature Package (Self-contained)
│   │   ├── handler.go               # HTTP handlers (CreateReview, ListByBook)
│   │   ├── service.go               # Business logic & event processing
│   │   ├── repository.go            # Storage interface + SQL implementation
│   │   └── model.go                 # Data structs (DB schema + JSON tags)
│   │
│   └── platform/                    # Shared infrastructure connections
│       ├── database/                # DB connection pool & migrations
│       └── broker/                  # Message broker client connection
│
├── go.mod
└── README.md
```

---

## 🚦 Ingress Layer 1: Centralized HTTP Router

All HTTP route bindings, path parameters, and global middlewares are consolidated into `internal/transport/http/router.go`. Feature packages never bind their own routes directly to global state.

```go
// internal/transport/http/router.go
package http

import (
    "net/http"
    "ExampleMicroservice/internal/review"
)

type RouterConfig struct {
    ReviewHandler *review.Handler
}

func NewRouter(cfg RouterConfig) http.Handler {
    mux := http.NewServeMux() // Go 1.22+ routing or Chi

    // Global Middlewares
    handler := LoggingMiddleware(RecoveryMiddleware(mux))

    // Health Check
    mux.HandleFunc("GET /healthz", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
    })

    // Feature Routes
    mux.HandleFunc("POST /api/v1/reviews", cfg.ReviewHandler.CreateReview)
    mux.HandleFunc("GET /api/v1/books/{id}/reviews", cfg.ReviewHandler.ListByBook)

    return handler
}
```

---

## 📨 Ingress Layer 2: Asynchronous Event Consumer

Satellite services consume events published by the monolith (e.g. `BookBorrowed`, `BookReturned`) asynchronously to avoid synchronous distributed coupling.

```go
// internal/transport/event/consumer.go
package event

import (
    "context"
    "log"
    "ExampleMicroservice/internal/platform/broker"
    "ExampleMicroservice/internal/review"
)

type Consumer struct {
    client  *broker.Client
    service *review.Service
}

func NewConsumer(client *broker.Client, svc *review.Service) *Consumer {
    return &Consumer{client: client, service: svc}
}

func (c *Consumer) Start(ctx context.Context) {
    messages := c.client.Subscribe(ctx, "library.events")

    for {
        select {
        case <-ctx.Done():
            log.Println("Event consumer shutting down...")
            return
        case msg := <-messages:
            if err := c.handleEvent(ctx, msg); err != nil {
                log.Printf("Failed to process event: %v", err)
            }
        }
    }
}

func (c *Consumer) handleEvent(ctx context.Context, msg broker.Message) error {
    switch msg.Type {
    case "BookBorrowed":
        return c.service.HandleBookBorrowedEvent(ctx, msg.Payload)
    default:
        return nil // Ignore irrelevant events
    }
}
```

---

## 🔌 Composition Root: `cmd/api/main.go`

`main.go` orchestrates the lifecycle, dependency injection, and clean graceful shutdown of both the HTTP server and the background event consumer:

```go
// cmd/api/main.go
package main

import (
    "context"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    transportHttp "ExampleMicroservice/internal/transport/http"
    "ExampleMicroservice/internal/transport/event"
    "ExampleMicroservice/internal/review"
    "ExampleMicroservice/internal/platform/database"
    "ExampleMicroservice/internal/platform/broker"
)

func main() {
    ctx, cancel := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    defer cancel()

    // 1. Initialize Infrastructure
    db := database.MustConnect(os.Getenv("DATABASE_URL"))
    defer db.Close()

    brokerClient := broker.MustConnect(os.Getenv("BROKER_URL"))
    defer brokerClient.Close()

    // 2. Initialize Feature Components
    reviewRepo := review.NewPostgresRepository(db)
    reviewSvc := review.NewService(reviewRepo)
    reviewHandler := review.NewHandler(reviewSvc)

    // 3. Centralized HTTP Router
    router := transportHttp.NewRouter(transportHttp.RouterConfig{
        ReviewHandler: reviewHandler,
    })

    server := &http.Server{
        Addr:    ":8081",
        Handler: router,
    }

    // 4. Start Background Event Worker
    eventConsumer := event.NewConsumer(brokerClient, reviewSvc)
    go eventConsumer.Start(ctx)

    // 5. Start HTTP Server
    go func() {
        log.Println("Microservice HTTP server running on :8081")
        if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("HTTP server error: %v", err)
        }
    }()

    // 6. Graceful Shutdown
    <-ctx.Done()
    shutdownCtx, shutdownCancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer shutdownCancel()

    if err := server.Shutdown(shutdownCtx); err != nil {
        log.Printf("Server shutdown forced: %v", err)
    }
    log.Println("Microservice stopped cleanly")
}
```

---

## 📦 Code Sharing Strategy (DRY vs. Coupling)

Follow Rob Pike’s Go proverb: **"A little copying is better than a little dependency."**

### ❌ What to Duplicate (Never put in a shared package):
- **Domain models, DTOs, and event payloads**: If Service A and Service B share a common domain library, a change in Service A can break Service B at build-time. Duplicate small event definitions and domain structs.

### ✅ What to Share (Cross-cutting plumbing only):
- **Infrastructure utilities**: Generic logging wrappers (`pkg/logger`), JWT authentication middleware (`pkg/auth`), and standard HTTP response helpers (`pkg/httputil`).
- **Rule of 3**: Do not extract utility code into a shared repository until at least 3 services need the exact same implementation.
