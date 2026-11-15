# Clean Architecture Example - Library System

This project demonstrates Clean Architecture principles in Go using a Library Management System as a concrete blueprint for modular monoliths.

---

## 🏗️ Architectural Overview

This architecture follows the **Dependency Inversion Principle**:
- **Source code dependencies point inward only.** Inner layers know nothing about outer layers.
- **Application defines, Infrastructure adapts.** The application layer defines the interfaces (ports) it needs, and the infrastructure layer provides the implementations (adapters).
- **Domain entities are the core.** Use cases and repository contracts operate directly on rich `domain/entities`. Redundant application-level DTOs are omitted to prevent anemic domain models and excessive mapping boilerplate.

```
┌─────────────────────────────────────────────────────────────┐
│                 Interfaces Layer (Driving)                  │
│             HTTP Handlers, API DTOs, CLI, Routers           │
└──────────────────────────────┬──────────────────────────────┘
                               │ depends on ↓
┌──────────────────────────────▼──────────────────────────────┐
│                  Application Layer (Core)                   │
│   Use Cases (Orchestration)                                 │
│   Repository Interfaces (Outbound Ports)                    │
│   Service Interfaces    (Outbound Ports)                    │
└──────────────────────────────┬──────────────────────────────┘
                               │ depends on ↓
┌──────────────────────────────▼──────────────────────────────┐
│                    Domain Layer (Center)                    │
│   Entities & Aggregates (State + Business Invariants)       │
│                     ← ZERO DEPENDENCIES →                   │
└──────────────────────────────▲──────────────────────────────┘
                               │ implements / adapts to
┌──────────────────────────────┴──────────────────────────────┐
│                Infrastructure Layer (Driven)                │
│   PostgreSQL Repositories, Database Records                 │
│   Third-Party API Clients (Loyalty, Serasa, Clearing)       │
└─────────────────────────────────────────────────────────────┘
```

---

## 🧭 Architectural Strategy: Monolith vs. Satellite Microservices

We follow a **Monolith First** engineering strategy. Distributed systems add networking, latency, and consistency overhead; starting with a well-structured modular monolith keeps shipping speed high and operational costs low.

| When to keep it in the Monolith | When to extract a Satellite Microservice |
| :--- | :--- |
| Core business domains (e.g. Clients, Books, Borrowing) | Distinct sub-domains / Bounded Contexts (e.g. Reviews, Notifications) |
| Requires strong ACID transactions and immediate consistency | Can operate asynchronously via domain events (Eventual Consistency) |
| Managed by a single team with shared release cycles | Independent scaling requirements or failure blast-radius containment |

> [!TIP]
> **Satellite Microservice Standard**: For the architecture layout, centralized routing, and asynchronous event consumer patterns of satellite services, see **[MICROSERVICES.md](file:///home/jorgesarkis/Wintap/CleanExampleGo/MICROSERVICES.md)**.

---

## 📝 Architectural Decision Record (ADR): Domain Entities vs Application DTOs

### Context
In layered architectures, an application can either:
1. Define intermediate **Application DTOs** for repository signatures and use case inputs/outputs, treating the domain layer as an optional calculation helper.
2. Define repository and use case contracts directly in terms of **Domain Entities**, keeping rich business logic encapsulated in entities.

### Decision
We use **Domain Entities directly in Application contracts** and eliminate `application/dto/`.

### Rationale
- **Prevents Anemic Domain Models**: Passing flat DTOs forces use cases to handle entity validation procedurally (e.g., checking book borrowing limits manually inside use cases). Working with Domain Entities preserves OOP/DDD encapsulation (e.g., calling `client.CanBorrow()`).
- **Eliminates Quadruple Model Duplication**: Using Application DTOs requires maintaining 4 distinct representations of the same concept (`ClientRecord` in DB, `ClientDTO` in Application, `Client` in Domain, `ClientResponse` in API) and writing 3 distinct mapper functions per entity.
- **Clear Two-Boundary Mapping**:
  - **Outer / Delivery Boundary**: `interfaces/dto` (API DTOs) ↔ Use Case parameters / Domain Entities.
  - **Data Storage Boundary**: `infrastructure/*/entities` (DB Records) ↔ `domain/entities` (Domain Entities).

---

## 📁 Project Structure

```
CleanExampleGo/
├── domain/                          # Enterprise Business Rules (The Heart)
│   └── entities/                    # Rich entities with business invariants & logic
│       ├── client.go                # Client entity (logic: CanBorrow, AddBook, etc.)
│       └── book.go                  # Book entity (logic: Rent, Return, IsOverdue)
│
├── application/                     # Application Business Rules (Orchestration)
│   ├── repositories/                # Outbound repository interfaces (Contracts)
│   │   ├── client_repository.go     # Client data access port
│   │   └── book_repository.go       # Book data access port
│   │
│   ├── services/                    # Outbound external service interfaces (Contracts)
│   │   ├── promotion_service.go     # Promotion client port
│   │   └── points_service.go        # Loyalty points client port
│   │
│   └── usecases/                    # Inbound use cases (Application workflows)
│       ├── rent_book.go             # Rent book orchestration
│       ├── return_book.go           # Return book orchestration
│       └── check_promotions.go      # Promotion check orchestration
│
├── infrastructure/                  # Frameworks, Drivers & Adapters
│   ├── postgresql/                  # PostgreSQL implementation
│   │   ├── entities/                # DB-specific records/tables (Internal)
│   │   │   ├── client_record.go     # Maps DB table ↔ domain/entities.Client
│   │   │   └── book_record.go       # Maps DB table ↔ domain/entities.Book
│   │   ├── repositories/            # Implements application/repositories
│   │   │   ├── client_repository.go
│   │   │   └── book_repository.go
│   │   └── connection.go            # SQL connection pooling
│   │
│   ├── loyalty_points/              # External Loyalty API client adapter
│   │   ├── entities/                # External API response payloads (Internal)
│   │   ├── client.go                # Raw HTTP client
│   │   └── service_impl.go          # Implements application/services.PointsService
│   │
│   └── http/                        # Shared HTTP utilities (retries, timeouts)
│       └── client.go
│
├── interfaces/                      # Driving Adapters (Delivery Mechanisms)
│   ├── api/                         # HTTP Handlers & Routing
│   │   ├── handlers/
│   │   │   ├── book_handler.go
│   │   │   └── client_handler.go
│   │   └── router.go
│   │
│   └── dto/                         # Delivery-specific DTOs (Request / Response)
│       ├── book_dto.go              # RentBookRequest, RentBookResponse
│       └── client_dto.go            # ClientResponse
│
├── cmd/                             # Application Entrypoints
│   └── api/
│       └── main.go                  # Composition Root (Dependency Injection wiring)
│
├── go.mod
└── README.md
```

---

## 🧩 Layer Responsibilities & Examples

### 1. Domain Layer (`domain/`)
The domain layer holds core business rules and state. It has **zero imports from other layers and zero external dependencies**.

```go
// domain/entities/client.go
package entities

import "errors"

type Client struct {
    ID              string
    Name            string
    Email           string
    Points          int
    BorrowedBookIDs []string
    MaxBooksAllowed int
}

// Business rule: Enforce loan eligibility
func (c *Client) CanBorrow() bool {
    return len(c.BorrowedBookIDs) < c.MaxBooksAllowed && c.Points >= 0
}

func (c *Client) BorrowBook(bookID string) error {
    if !c.CanBorrow() {
        return errors.New("client borrowing limit reached or insufficient points")
    }
    c.BorrowedBookIDs = append(c.BorrowedBookIDs, bookID)
    return nil
}

func (c *Client) ReturnBook(bookID string) {
    var updated []string
    for _, id := range c.BorrowedBookIDs {
        if id != bookID {
            updated = append(updated, id)
        }
    }
    c.BorrowedBookIDs = updated
}
```

```go
// domain/entities/book.go
package entities

import "time"

type Book struct {
    ID          string
    Title       string
    Author      string
    IsAvailable bool
    BorrowedBy  string
    DueDate     *time.Time
}

func (b *Book) Rent(clientID string, duration time.Duration) {
    b.IsAvailable = false
    b.BorrowedBy = clientID
    due := time.Now().Add(duration)
    b.DueDate = &due
}

func (b *Book) Return() {
    b.IsAvailable = true
    b.BorrowedBy = ""
    b.DueDate = nil
}

func (b *Book) IsOverdue() bool {
    if b.DueDate == nil {
        return false
    }
    return time.Now().After(*b.DueDate)
}
```

---

### 2. Application Layer (`application/`)
The application layer orchestrates use cases. It defines the contracts (ports) that infrastructure must fulfill.

#### Repository Interface (Port):
```go
// application/repositories/client_repository.go
package repositories

import (
    "context"
    "ExampleProject/domain/entities"
)

type ClientRepository interface {
    GetByID(ctx context.Context, id string) (*entities.Client, error)
    Update(ctx context.Context, client *entities.Client) error
    Save(ctx context.Context, client *entities.Client) error
}
```

#### Service Interface (Port):
```go
// application/services/points_service.go
package services

import "context"

type PointsService interface {
    AddPoints(ctx context.Context, clientID string, points int) error
    ConsumePoints(ctx context.Context, clientID string, points int) error
}
```

#### Use Case Implementation:
```go
// application/usecases/rent_book.go
package usecases

import (
    "context"
    "errors"
    "time"
    "ExampleProject/application/repositories"
    "ExampleProject/application/services"
)

type RentBookUseCase struct {
    clientRepo    repositories.ClientRepository
    bookRepo      repositories.BookRepository
    promotionSvc  services.PromotionService
    pointsService services.PointsService
}

func NewRentBookUseCase(
    clientRepo repositories.ClientRepository,
    bookRepo repositories.BookRepository,
    promotionSvc services.PromotionService,
    pointsService services.PointsService,
) *RentBookUseCase {
    return &RentBookUseCase{
        clientRepo:    clientRepo,
        bookRepo:      bookRepo,
        promotionSvc:  promotionSvc,
        pointsService: pointsService,
    }
}

func (uc *RentBookUseCase) Execute(ctx context.Context, clientID, bookID string, days int) error {
    client, err := uc.clientRepo.GetByID(ctx, clientID)
    if err != nil {
        return err
    }

    if !client.CanBorrow() {
        return errors.New("client cannot borrow more books")
    }

    book, err := uc.bookRepo.GetByID(ctx, bookID)
    if err != nil {
        return err
    }

    if !book.IsAvailable {
        return errors.New("book is currently rented")
    }

    // Apply domain operations
    if err := client.BorrowBook(bookID); err != nil {
        return err
    }
    book.Rent(clientID, time.Duration(days)*24*time.Hour)

    // Optional loyalty promotion orchestration
    hasPromo, _ := uc.promotionSvc.HasActivePromotion(ctx, clientID)
    if hasPromo {
        _ = uc.pointsService.AddPoints(ctx, clientID, 10)
    }

    // Persist changes
    if err := uc.clientRepo.Update(ctx, client); err != nil {
        return err
    }
    return uc.bookRepo.Update(ctx, book)
}
```

---

### 3. Infrastructure Layer (`infrastructure/`)
Contains technology-specific implementations. Translates external models (database rows, HTTP payloads) to and from `domain/entities`.

#### Database Entity (Internal to Infrastructure):
```go
// infrastructure/postgresql/entities/client_record.go
package entities

import (
    "strings"
    "time"
    "ExampleProject/domain/entities"
)

type ClientRecord struct {
    ID              string
    Name            string
    Email           string
    Points          int
    BorrowedBookIDs string // comma-separated in SQL
    MaxBooksAllowed int
    CreatedAt       time.Time
    UpdatedAt       time.Time
}

func (r *ClientRecord) ToDomain() *entities.Client {
    var books []string
    if r.BorrowedBookIDs != "" {
        books = strings.Split(r.BorrowedBookIDs, ",")
    }
    return &entities.Client{
        ID:              r.ID,
        Name:            r.Name,
        Email:           r.Email,
        Points:          r.Points,
        BorrowedBookIDs: books,
        MaxBooksAllowed: r.MaxBooksAllowed,
    }
}

func FromDomainClient(c *entities.Client) *ClientRecord {
    return &ClientRecord{
        ID:              c.ID,
        Name:            c.Name,
        Email:           c.Email,
        Points:          c.Points,
        BorrowedBookIDs: strings.Join(c.BorrowedBookIDs, ","),
        MaxBooksAllowed: c.MaxBooksAllowed,
        UpdatedAt:       time.Now(),
    }
}
```

#### Repository Implementation (Adapter):
```go
// infrastructure/postgresql/repositories/client_repository.go
package repositories

import (
    "context"
    "database/sql"
    "ExampleProject/application/repositories"
    "ExampleProject/domain/entities"
    pgEntities "ExampleProject/infrastructure/postgresql/entities"
)

type PostgreSQLClientRepository struct {
    db *sql.DB
}

func NewPostgreSQLClientRepository(db *sql.DB) repositories.ClientRepository {
    return &PostgreSQLClientRepository{db: db}
}

func (r *PostgreSQLClientRepository) GetByID(ctx context.Context, id string) (*entities.Client, error) {
    query := `SELECT id, name, email, points, borrowed_books, max_books FROM clients WHERE id = $1`
    var rec pgEntities.ClientRecord
    err := r.db.QueryRowContext(ctx, query, id).Scan(
        &rec.ID, &rec.Name, &rec.Email, &rec.Points, &rec.BorrowedBookIDs, &rec.MaxBooksAllowed,
    )
    if err != nil {
        return nil, err
    }
    return rec.ToDomain(), nil
}

func (r *PostgreSQLClientRepository) Update(ctx context.Context, client *entities.Client) error {
    rec := pgEntities.FromDomainClient(client)
    query := `UPDATE clients SET points = $1, borrowed_books = $2, updated_at = $3 WHERE id = $4`
    _, err := r.db.ExecContext(ctx, query, rec.Points, rec.BorrowedBookIDs, rec.UpdatedAt, rec.ID)
    return err
}

func (r *PostgreSQLClientRepository) Save(ctx context.Context, client *entities.Client) error {
    rec := pgEntities.FromDomainClient(client)
    query := `INSERT INTO clients (id, name, email, points, borrowed_books, max_books, created_at, updated_at)
              VALUES ($1, $2, $3, $4, $5, $6, $7, $8)`
    _, err := r.db.ExecContext(ctx, query, rec.ID, rec.Name, rec.Email, rec.Points, rec.BorrowedBookIDs, rec.MaxBooksAllowed, rec.CreatedAt, rec.UpdatedAt)
    return err
}
```

---

### 4. Interfaces Layer (`interfaces/`)
Primary driving adapter. Receives external requests, parses them into API DTOs, validates formats, calls use cases, and serializes responses.

#### API DTO:
```go
// interfaces/dto/book_dto.go
package dto

type RentBookRequest struct {
    ClientID string `json:"client_id"`
    BookID   string `json:"book_id"`
    Days     int    `json:"days"`
}

type RentBookResponse struct {
    Success bool   `json:"success"`
    Message string `json:"message"`
}
```

#### API Handler:
```go
// interfaces/api/handlers/book_handler.go
package handlers

import (
    "encoding/json"
    "net/http"
    "ExampleProject/application/usecases"
    "ExampleProject/interfaces/dto"
)

type BookHandler struct {
    rentBookUseCase *usecases.RentBookUseCase
}

func NewBookHandler(rentBookUseCase *usecases.RentBookUseCase) *BookHandler {
    return &BookHandler{rentBookUseCase: rentBookUseCase}
}

func (h *BookHandler) RentBook(w http.ResponseWriter, r *http.Request) {
    var req dto.RentBookRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "invalid request body", http.StatusBadRequest)
        return
    }

    if err := h.rentBookUseCase.Execute(r.Context(), req.ClientID, req.BookID, req.Days); err != nil {
        http.Error(w, err.Error(), http.StatusUnprocessableEntity)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(dto.RentBookResponse{
        Success: true,
        Message: "Book rented successfully",
    })
}
```

---

## 🔌 Composition Root (`cmd/api/main.go`)

Wiring is strictly performed at the composition root using constructor dependency injection:

```go
// cmd/api/main.go
package main

import (
    "database/sql"
    "log"
    "net/http"
    "os"

    "ExampleProject/application/usecases"
    "ExampleProject/infrastructure/loyalty_points"
    "ExampleProject/infrastructure/postgresql"
    pgRepos "ExampleProject/infrastructure/postgresql/repositories"
    "ExampleProject/infrastructure/promotion"
    "ExampleProject/interfaces/api/handlers"
)

func main() {
    // 1. Initialize Infrastructure dependencies
    db, err := sql.Open("postgres", os.Getenv("DATABASE_URL"))
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    // 2. Initialize Repositories (returns application/repositories interfaces)
    clientRepo := pgRepos.NewPostgreSQLClientRepository(db)
    bookRepo := pgRepos.NewPostgreSQLBookRepository(db)

    // 3. Initialize External Services (returns application/services interfaces)
    pointsSvc := loyalty_points.NewLoyaltyPointsService(os.Getenv("LOYALTY_URL"))
    promotionSvc := promotion.NewPromotionService(os.Getenv("PROMOTION_URL"))

    // 4. Initialize Use Cases (inject interfaces)
    rentBookUseCase := usecases.NewRentBookUseCase(clientRepo, bookRepo, promotionSvc, pointsSvc)

    // 5. Initialize Delivery Handlers
    bookHandler := handlers.NewBookHandler(rentBookUseCase)

    // 6. Register Routes & Start Server
    http.HandleFunc("/api/books/rent", bookHandler.RentBook)
    log.Println("Server running on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

---

## 🧪 Testing Strategy

1. **Unit Testing Domain Entities**: Zero mocks needed. Test pure Go methods (`client.CanBorrow()`, `book.IsOverdue()`) directly.
2. **Unit Testing Use Cases**: Mock the `application/repositories` and `application/services` interfaces. Verify business flow and state transitions on returned `domain/entities`.
3. **Integration Testing Infrastructure**: Test repository SQL queries against a real containerized database (e.g. Testcontainers).
