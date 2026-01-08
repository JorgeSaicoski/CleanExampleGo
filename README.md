# Clean Architecture Example - Library System

This project demonstrates Clean Architecture principles in Go using a Library Management System as an example.

## 📁 Project Structure

```
ExampleProject/
├── domain/                          # Enterprise Business Rules (OPTIONAL)
│   └── entities/                    # Pure business entities with logic
│       ├── client.go               # Client entity (optional)
│       ├── book.go                 # Book entity (optional)
│       └── promotion.go            # Promotion entity (optional)
│
├── application/                     # Application Business Rules (CORE)
│   ├── dto/                         # Application DTOs (data contracts)
│   │   ├── client_dto.go           # What application expects for clients
│   │   ├── book_dto.go             # What application expects for books
│   │   └── points_dto.go           # What application expects for points
│   │
│   ├── repositories/                # Repository interfaces (contracts)
│   │   ├── client_repository.go    # Client data access contract
│   │   └── book_repository.go      # Book data access contract
│   │
│   ├── services/                    # Service interfaces (contracts)
│   │   ├── promotion_service.go    # Promotion service interface
│   │   └── points_service.go       # Points service interface
│   │
│   └── usecases/                    # Use cases (orchestration)
│       ├── rent_book.go            # Rent book use case
│       ├── return_book.go          # Return book use case
│       └── check_promotions.go     # Check client promotions use case
│
├── infrastructure/                  # Frameworks & Drivers (ADAPTERS)
│   ├── postgresql/                  # PostgreSQL implementation
│   │   ├── entities/               # Database-specific entities (internal)
│   │   │   ├── client_record.go   # DB structure for clients
│   │   │   └── book_record.go     # DB structure for books
│   │   ├── repositories/           # Repository implementations
│   │   │   ├── client_repository.go  # Implements app/repositories interface
│   │   │   └── book_repository.go    # Implements app/repositories interface
│   │   └── migrations/             # SQL migrations
│   │
│   ├── loyalty_points/             # External loyalty API client
│   │   ├── entities/               # API response structures (internal)
│   │   ├── client.go               # HTTP client
│   │   └── service_impl.go         # Implements app/services interface
│   │
│   └── http/                        # Shared HTTP utilities
│       └── client.go
│
├── interfaces/                      # Interface Adapters (API LAYER)
│   ├── api/                         # HTTP handlers/controllers
│   │   ├── handlers/
│   │   │   ├── client_handler.go
│   │   │   └── book_handler.go
│   │   └── router.go
│   │
│   └── dto/                         # API DTOs (request/response)
│       ├── client_dto.go           # API client representation
│       └── book_dto.go             # API book representation
│
├── pkg/                             # Shared utilities
│   ├── errors/                      # Custom error types
│   │   └── errors.go
│   └── logger/                      # Logging utilities
│       └── logger.go
│
├── cmd/                             # Application entry points
│   └── api/
│       └── main.go                  # Main application
│
├── go.mod
└── README.md
```

## 🏗️ Architecture Layers

### Key Architectural Principle

**"Application Defines, Infrastructure Adapts"**

The **application layer** defines what it needs (interfaces and DTOs), and the **infrastructure layer** provides implementations. This follows the **Dependency Inversion Principle**.

```
┌─────────────────────────────────────────────┐
│         Application Layer (CORE)            │
│  ┌─────────────────────────────────────┐   │
│  │ Use Cases                           │   │
│  │  - Orchestrates business logic      │   │
│  │  - Uses DTOs and interfaces         │   │
│  └──────┬────────────────┬─────────────┘   │
│         │ depends on     │                  │
│  ┌──────▼──────────┐  ┌──▼─────────────┐   │
│  │ Repositories    │  │ Services       │   │
│  │ (interfaces)    │  │ (interfaces)   │   │
│  └─────────────────┘  └────────────────┘   │
│  ┌──────────────────────────────────────┐  │
│  │ DTOs (data contracts)                │  │
│  └──────────────────────────────────────┘  │
└──────────────────┬──────────────────────────┘
                   ↑ implements
┌──────────────────┴──────────────────────────┐
│      Infrastructure Layer (ADAPTERS)        │
│  ┌────────────────┐  ┌──────────────────┐  │
│  │ PostgreSQL     │  │ Loyalty Points   │  │
│  │ Repositories   │  │ Service          │  │
│  └────────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────┘
```

### 1. Domain Layer (Optional)
**Location:** `domain/`

Optional layer for pure business entities with business logic methods. Can be omitted for simple CRUD applications.

**Components:**
- **Entities** (`domain/entities/`): Core business objects with business rules

**Rules:**
- ✅ NO dependencies on other layers
- ✅ Pure Go code, no external libraries
- ✅ Contains business logic and validation
- ❌ NO database, HTTP, or framework code

**Example - Client Entity:**
```go
// domain/entities/client.go
package entities

type Client struct {
    ID              string
    Name            string
    Email           string
    Points          int
    BorrowedBooks   []string
    MaxBooksAllowed int
}

// Business rule: Client can borrow if under limit and has valid account
func (c *Client) CanBorrow() bool {
    return len(c.BorrowedBooks) < c.MaxBooksAllowed && c.Points >= 0
}

// Business rule: Check if client has minimum points
func (c *Client) HasMinimumPoints(required int) bool {
    return c.Points >= required
}
```

---

### 2. Application Layer (CORE)
**Location:** `application/`

The heart of your application. Defines what the application needs and contains business logic orchestration.

**Components:**
- **DTOs** (`application/dto/`): Data contracts - what the application expects
- **Repository Interfaces** (`application/repositories/`): Data access contracts
- **Service Interfaces** (`application/services/`): External service contracts  
- **Use Cases** (`application/usecases/`): Business logic orchestration

**Rules:**
- ✅ Defines interfaces and DTOs
- ✅ Orchestrates business logic
- ✅ Technology-agnostic (no mention of PostgreSQL, HTTP, etc.)
- ❌ NO implementation of repositories or services
- ❌ NO knowledge of HTTP, databases, or UI

**Example - Application DTO:**
```go
// application/dto/client_dto.go
package dto

type ClientDTO struct {
    ID              string
    Name            string
    Email           string
    Points          int
    MaxBooksAllowed int
    BorrowedBookIDs []string
}
```

**Example - Repository Interface:**
```go
// application/repositories/client_repository.go
package repositories

import "ExampleProject/application/dto"

type ClientRepository interface {
    GetByID(id string) (*dto.ClientDTO, error)
    Save(client *dto.ClientDTO) error
    Update(client *dto.ClientDTO) error
}
```

**Example - Use Case:**
```go
// application/usecases/rent_book.go
package usecases

import (
    "ExampleProject/application/dto"
    "ExampleProject/application/repositories"
    "ExampleProject/application/services"
    "errors"
)

type RentBookUseCase struct {
    clientRepo      repositories.ClientRepository
    bookRepo        repositories.BookRepository
    promotionSvc    services.PromotionService
    pointsService   services.PointsService
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

func (uc *RentBookUseCase) Execute(clientID, bookID string, durationDays int) error {
    // 1. Get client (returns DTO)
    client, err := uc.clientRepo.GetByID(clientID)
    if err != nil {
        return err
    }

    // 2. Check business rule
    if len(client.BorrowedBookIDs) >= client.MaxBooksAllowed {
        return errors.New("client cannot borrow more books")
    }

    // 3. Get book (returns DTO)
    book, err := uc.bookRepo.GetByID(bookID)
    if err != nil {
        return err
    }

    // 4. Check if available
    if !book.IsAvailable {
        return errors.New("book is not available")
    }

    // 5. Check for promotions (returns DTO)
    hasPromotion, err := uc.promotionSvc.HasActivePromotion(clientID)
    if err != nil {
        hasPromotion = false // Log error but continue
    }

    // 6. Update state
    book.IsAvailable = false
    book.BorrowedBy = clientID
    client.BorrowedBookIDs = append(client.BorrowedBookIDs, bookID)

    // 7. Award points if has promotion
    if hasPromotion {
        uc.pointsService.AddPoints(clientID, 10)
    }

    // 8. Persist changes
    uc.bookRepo.Update(book)
    uc.clientRepo.Update(client)

    return nil
}
```

---

### 3. Infrastructure Layer (ADAPTERS)
**Location:** `infrastructure/`

Implements the interfaces defined in the application layer. Adapts external systems to what the application expects.

**Components:**
- **PostgreSQL** (`infrastructure/postgresql/`): Database implementation
  - `entities/`: Database-specific structures (internal)
  - `repositories/`: Implements `application/repositories` interfaces
- **External Services** (`infrastructure/loyalty_points/`): API clients
  - `entities/`: API response structures (internal)
  - `service_impl.go`: Implements `application/services` interfaces

**Rules:**
- ✅ Implements application interfaces
- ✅ Converts between infrastructure entities and application DTOs
- ✅ Contains database, HTTP, and framework code
- ❌ Application never imports infrastructure code

**Example - PostgreSQL Repository:**
```go
// infrastructure/postgresql/repositories/client_repository.go
package repositories

import (
    "database/sql"
    "ExampleProject/application/dto"
    "ExampleProject/application/repositories"
    pgEntities "ExampleProject/infrastructure/postgresql/entities"
)

type PostgreSQLClientRepository struct {
    db *sql.DB
}

// Implements application/repositories.ClientRepository
func NewPostgreSQLClientRepository(db *sql.DB) repositories.ClientRepository {
    return &PostgreSQLClientRepository{db: db}
}

func (r *PostgreSQLClientRepository) GetByID(id string) (*dto.ClientDTO, error) {
    var record pgEntities.ClientRecord
    err := r.db.QueryRow("SELECT ... WHERE id = $1", id).Scan(...)
    if err != nil {
        return nil, err
    }
    
    // Convert infrastructure entity to application DTO
    return record.ToDTO(), nil
}
```

**Example - Infrastructure Entity (Internal):**
```go
// infrastructure/postgresql/entities/client_record.go
package entities

import (
    "time"
    "ExampleProject/application/dto"
)

// ClientRecord is the database representation (internal to infrastructure)
type ClientRecord struct {
    ID              string
    Name            string
    Email           string
    Points          int
    MaxBooksAllowed int
    CreatedAt       time.Time  // Database-specific
    UpdatedAt       time.Time  // Database-specific
}

// Convert to application DTO
func (r *ClientRecord) ToDTO() *dto.ClientDTO {
    return &dto.ClientDTO{
        ID:              r.ID,
        Name:            r.Name,
        Email:           r.Email,
        Points:          r.Points,
        MaxBooksAllowed: r.MaxBooksAllowed,
    }
}
```

---

### 4. Interfaces Layer (API)
**Location:** `interfaces/`

Handles external communication (HTTP, gRPC, CLI, etc.). Converts between API formats and application DTOs.

**Components:**
- **API Handlers** (`interfaces/api/handlers/`): HTTP request handlers
- **API DTOs** (`interfaces/dto/`): Request/response structures for API

**Rules:**
- ✅ Handles HTTP requests/responses
- ✅ Converts between API DTOs and application DTOs
- ✅ Calls use cases
- ❌ NO business logic

**Example - API Handler:**
```go
// interfaces/api/handlers/client_handler.go
package handlers

import (
    "net/http"
    "ExampleProject/application/usecases"
    apiDTO "ExampleProject/interfaces/dto"
)

type ClientHandler struct {
    rentBookUseCase usecases.RentBookUseCase
}

func (h *ClientHandler) RentBook(w http.ResponseWriter, r *http.Request) {
    // 1. Parse API request
    var req apiDTO.RentBookRequest
    json.NewDecoder(r.Body).Decode(&req)
    
    // 2. Call use case
    err := h.rentBookUseCase.Execute(req.ClientID, req.BookID, req.Duration)
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    
    // 3. Return API response
    resp := apiDTO.RentBookResponse{Success: true}
    json.NewEncoder(w).Encode(resp)
}
```

---

## 🔄 Data Flow Example

### Renting a Book Request Flow

```
1. HTTP Request arrives
   ↓
2. Handler (interfaces/api/handlers)
   - Parses API DTO
   ↓
3. Use Case (application/usecases)
   - Uses application DTOs
   - Calls repository interface
   ↓
4. Repository Implementation (infrastructure/postgresql)
   - Queries database
   - Converts database record to application DTO
   - Returns application DTO
   ↓
5. Use Case continues
   - Applies business logic
   - Calls service interface
   ↓
6. Service Implementation (infrastructure/loyalty_points)
   - Calls external API
   - Converts API response to application DTO
   - Returns application DTO
   ↓
7. Use Case finishes
   - Returns result to handler
   ↓
8. Handler
   - Converts to API DTO
   - Returns HTTP response
```

### Key Point: DTOs at Each Layer

- **API DTOs** (`interfaces/dto/`): What external clients see
- **Application DTOs** (`application/dto/`): What the application works with
- **Infrastructure Entities** (internal): How external systems represent data

The infrastructure and interface layers convert between these formats.

---

## 🎯 Dependency Rule

**The Dependency Rule:** Source code dependencies must point inward only.

```
Interfaces (API) ────depends on───→ Application
Infrastructure   ────implements───→ Application (interfaces)
```

- Outer layers depend on inner layers
- Inner layers know nothing about outer layers
- Application defines interfaces, infrastructure implements them
- This makes the application independent of frameworks, databases, and UIs

---

## 🚀 Getting Started

### Prerequisites
- Go 1.21+
- PostgreSQL (for database implementation)

### Setup

1. Clone the repository
```bash
git clone <repository-url>
cd ExampleProject
```

2. Install dependencies
```bash
go mod download
```

3. Run the application
```bash
go run cmd/api/main.go
```

---

## 📚 Key Benefits

1. **Testability**: Easy to test with mocks for interfaces
2. **Flexibility**: Swap PostgreSQL for MongoDB without changing application code
3. **Independence**: Business logic doesn't depend on frameworks or databases
4. **Maintainability**: Changes are isolated to specific layers
5. **Clarity**: Clear separation of concerns and responsibilities

---

## 📖 Further Reading

Each folder contains its own README with detailed explanations and examples:
- `application/dto/README.md` - Application DTOs explained
- `application/repositories/README.md` - Repository interfaces
- `application/services/README.md` - Service interfaces
- `application/usecases/README.md` - Use case patterns
- `infrastructure/README.md` - Infrastructure implementations
- `infrastructure/postgresql/README.md` - PostgreSQL specific docs

---

**Remember**: Application defines what it needs (interfaces and DTOs), infrastructure provides implementations. This is the key to Clean Architecture!
    book, err := uc.bookRepo.GetByID(bookID)
    if err != nil {
        return err
    }

    if !book.IsAvailable {
        return errors.New("book is not available")
    }

    // 5. Add points if has promotion
    if hasPromotion {
        err = uc.pointsService.AddPoints(clientID, 10)
        if err != nil {
            return err
        }
    }

    // 6. Update entities
    client.BorrowedBooks = append(client.BorrowedBooks, bookID)
    book.IsAvailable = false

    // 7. Persist changes
    if err := uc.clientRepo.Update(client); err != nil {
        return err
    }

    if err := uc.bookRepo.Update(book); err != nil {
        return err
    }

    return nil
}
```

**Example - Service Interface:**
```go
// application/services/promotion_service.go
package services

type PromotionService interface {
    HasActivePromotion(clientID string) (bool, error)
    GetClientPromotions(clientID string) ([]string, error)
}

// application/services/points_service.go
package services

type PointsService interface {
    GetClientPoints(clientID string) (int, error)
    AddPoints(clientID string, points int) error
    ConsumePoints(clientID string, points int) error
}
```

---

### 3. Infrastructure Layer
**Location:** `infrastructure/`

Contains implementations of interfaces defined in inner layers. Deals with external systems.

**Components:**
- **Repository Implementations** (`infrastructure/repositories/`)
- **Service Implementations** (`infrastructure/services/`)
- **External HTTP Clients** (`infrastructure/http/`)

**Rules:**
- ✅ Implements interfaces from domain/application layers
- ✅ Contains database, API, and external service code
- ✅ Can depend on external libraries

**Example - Repository Implementation:**
```go
// infrastructure/repositories/client_repository_impl.go
package repositories

import (
    "ExampleProject/domain/entities"
    "database/sql"
    "errors"
)

type ClientRepositoryImpl struct {
    db *sql.DB
}

func NewClientRepository(db *sql.DB) *ClientRepositoryImpl {
    return &ClientRepositoryImpl{db: db}
}

func (r *ClientRepositoryImpl) GetByID(id string) (*entities.Client, error) {
    var client entities.Client
    
    query := `SELECT id, name, email, points, max_books_allowed FROM clients WHERE id = ?`
    err := r.db.QueryRow(query, id).Scan(
        &client.ID,
        &client.Name,
        &client.Email,
        &client.Points,
        &client.MaxBooksAllowed,
    )
    
    if err != nil {
        if err == sql.ErrNoRows {
            return nil, errors.New("client not found")
        }
        return nil, err
    }

    // Load borrowed books
    client.BorrowedBooks, err = r.getBorrowedBooks(id)
    if err != nil {
        return nil, err
    }

    return &client, nil
}

func (r *ClientRepositoryImpl) Save(client *entities.Client) error {
    query := `INSERT INTO clients (id, name, email, points, max_books_allowed) 
              VALUES (?, ?, ?, ?, ?)`
    _, err := r.db.Exec(query, client.ID, client.Name, client.Email, 
                        client.Points, client.MaxBooksAllowed)
    return err
}

func (r *ClientRepositoryImpl) Update(client *entities.Client) error {
    query := `UPDATE clients SET name = ?, email = ?, points = ?, 
              max_books_allowed = ? WHERE id = ?`
    _, err := r.db.Exec(query, client.Name, client.Email, client.Points, 
                        client.MaxBooksAllowed, client.ID)
    return err
}

func (r *ClientRepositoryImpl) getBorrowedBooks(clientID string) ([]string, error) {
    query := `SELECT book_id FROM borrowed_books WHERE client_id = ?`
    rows, err := r.db.Query(query, clientID)
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    var books []string
    for rows.Next() {
        var bookID string
        if err := rows.Scan(&bookID); err != nil {
            return nil, err
        }
        books = append(books, bookID)
    }

    return books, nil
}
```

**Example - External Service Implementation:**
```go
// infrastructure/services/promotion_service_impl.go
package services

import (
    "encoding/json"
    "fmt"
    "net/http"
)

type PromotionServiceImpl struct {
    baseURL string
    client  *http.Client
}

func NewPromotionService(baseURL string) *PromotionServiceImpl {
    return &PromotionServiceImpl{
        baseURL: baseURL,
        client:  &http.Client{},
    }
}

func (s *PromotionServiceImpl) HasActivePromotion(clientID string) (bool, error) {
    url := fmt.Sprintf("%s/api/promotions/active/%s", s.baseURL, clientID)
    
    resp, err := s.client.Get(url)
    if err != nil {
        return false, err
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        return false, nil
    }

    var result struct {
        HasPromotion bool `json:"has_promotion"`
    }

    if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
        return false, err
    }

    return result.HasPromotion, nil
}

func (s *PromotionServiceImpl) GetClientPromotions(clientID string) ([]string, error) {
    url := fmt.Sprintf("%s/api/promotions/client/%s", s.baseURL, clientID)
    
    resp, err := s.client.Get(url)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    var promotions []string
    if err := json.NewDecoder(resp.Body).Decode(&promotions); err != nil {
        return nil, err
    }

    return promotions, nil
}

// infrastructure/services/points_service_impl.go
package services

import (
    "bytes"
    "encoding/json"
    "fmt"
    "net/http"
)

type PointsServiceImpl struct {
    baseURL string
    client  *http.Client
}

func NewPointsService(baseURL string) *PointsServiceImpl {
    return &PointsServiceImpl{
        baseURL: baseURL,
        client:  &http.Client{},
    }
}

func (s *PointsServiceImpl) GetClientPoints(clientID string) (int, error) {
    url := fmt.Sprintf("%s/api/points/%s", s.baseURL, clientID)
    
    resp, err := s.client.Get(url)
    if err != nil {
        return 0, err
    }
    defer resp.Body.Close()

    var result struct {
        Points int `json:"points"`
    }

    if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
        return 0, err
    }

    return result.Points, nil
}

func (s *PointsServiceImpl) AddPoints(clientID string, points int) error {
    url := fmt.Sprintf("%s/api/points/add", s.baseURL)
    
    payload := map[string]interface{}{
        "client_id": clientID,
        "points":    points,
    }

    jsonData, err := json.Marshal(payload)
    if err != nil {
        return err
    }

    resp, err := s.client.Post(url, "application/json", bytes.NewBuffer(jsonData))
    if err != nil {
        return err
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        return fmt.Errorf("failed to add points: status %d", resp.StatusCode)
    }

    return nil
}

func (s *PointsServiceImpl) ConsumePoints(clientID string, points int) error {
    url := fmt.Sprintf("%s/api/points/consume", s.baseURL)
    
    payload := map[string]interface{}{
        "client_id": clientID,
        "points":    points,
    }

    jsonData, err := json.Marshal(payload)
    if err != nil {
        return err
    }

    resp, err := s.client.Post(url, "application/json", bytes.NewBuffer(jsonData))
    if err != nil {
        return err
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        return fmt.Errorf("failed to consume points: status %d", resp.StatusCode)
    }

    return nil
}
```

---

### 4. Interface Adapters Layer
**Location:** `interfaces/`

Converts data between the format most convenient for use cases/entities and the format most convenient for external agencies (web, CLI, etc.).

**Components:**
- **API Handlers** (`interfaces/api/handlers/`)
- **DTOs** (`interfaces/dto/`)
- **Router** (`interfaces/api/router.go`)

**Example - Handler:**
```go
// interfaces/api/handlers/book_handler.go
package handlers

import (
    "ExampleProject/application/usecases"
    "ExampleProject/interfaces/dto"
    "encoding/json"
    "net/http"
)

type BookHandler struct {
    rentBookUseCase *usecases.RentBookUseCase
}

func NewBookHandler(rentBookUseCase *usecases.RentBookUseCase) *BookHandler {
    return &BookHandler{
        rentBookUseCase: rentBookUseCase,
    }
}

func (h *BookHandler) RentBook(w http.ResponseWriter, r *http.Request) {
    var req dto.RentBookRequest
    
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Invalid request body", http.StatusBadRequest)
        return
    }

    // Call use case
    err := h.rentBookUseCase.Execute(req.ClientID, req.BookID)
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    // Return success response
    resp := dto.RentBookResponse{
        Success: true,
        Message: "Book rented successfully",
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(resp)
}
```

**Example - DTOs:**
```go
// interfaces/dto/book_dto.go
package dto

type RentBookRequest struct {
    ClientID string `json:"client_id"`
    BookID   string `json:"book_id"`
}

type RentBookResponse struct {
    Success bool   `json:"success"`
    Message string `json:"message"`
}

// interfaces/dto/client_dto.go
package dto

type ClientResponse struct {
    ID              string   `json:"id"`
    Name            string   `json:"name"`
    Email           string   `json:"email"`
    Points          int      `json:"points"`
    BorrowedBooks   []string `json:"borrowed_books"`
    CanBorrow       bool     `json:"can_borrow"`
}
```

---

## 🔄 Dependency Flow

```
┌─────────────────────────────────────────────────────┐
│                   Interfaces                        │
│              (HTTP Handlers, CLI)                   │
└─────────────────┬───────────────────────────────────┘
                  │ depends on ↓
┌─────────────────▼───────────────────────────────────┐
│                 Application                         │
│            (Use Cases, Services)                    │
└─────────────────┬───────────────────────────────────┘
                  │ depends on ↓
┌─────────────────▼───────────────────────────────────┐
│                   Domain                            │
│         (Entities, Repositories)                    │
│            ← NO DEPENDENCIES →                      │
└─────────────────────────────────────────────────────┘
                  ▲ implements
┌─────────────────┴───────────────────────────────────┐
│              Infrastructure                         │
│       (DB, External APIs, Frameworks)               │
└─────────────────────────────────────────────────────┘
```

**Key Principle:** Dependencies point INWARD. Inner layers know nothing about outer layers.

---

## 🚀 How to Use This Structure

### 1. Adding a New Feature

**Example: Add "Reserve Book" feature**

1. **Start with Domain:**
   - Add business rule to `domain/entities/book.go`:
     ```go
     func (b *Book) CanBeReserved() bool {
         return !b.IsAvailable && b.ReservationCount < 3
     }
     ```

2. **Create Use Case:**
   - Add `application/usecases/reserve_book.go`
   - Inject required repositories and services
   - Implement business logic

3. **Add Handler:**
   - Add `interfaces/api/handlers/book_handler.go::ReserveBook()`
   - Create DTOs for request/response

4. **Wire Everything in Main:**
   - Instantiate dependencies
   - Inject into use case
   - Register handler route

### 2. Adding External Service Integration

**Example: Add Email Service**

1. **Define Interface in Application:**
   ```go
   // application/services/email_service.go
   type EmailService interface {
       SendEmail(to, subject, body string) error
   }
   ```

2. **Implement in Infrastructure:**
   ```go
   // infrastructure/services/email_service_impl.go
   type EmailServiceImpl struct {
       smtpHost string
       smtpPort int
   }
   ```

3. **Inject into Use Case:**
   ```go
   type RentBookUseCase struct {
       // ...existing fields
       emailService services.EmailService
   }
   ```

---

## 📋 Complete Use Case Examples

### Use Case: Check Client Promotions

```go
// application/usecases/check_promotions.go
package usecases

import (
    "ExampleProject/domain/repositories"
    "ExampleProject/application/services"
)

type CheckPromotionsUseCase struct {
    clientRepo    repositories.ClientRepository
    promotionSvc  services.PromotionService
}

func NewCheckPromotionsUseCase(
    clientRepo repositories.ClientRepository,
    promotionSvc services.PromotionService,
) *CheckPromotionsUseCase {
    return &CheckPromotionsUseCase{
        clientRepo:   clientRepo,
        promotionSvc: promotionSvc,
    }
}

type PromotionInfo struct {
    ClientName   string
    Promotions   []string
    HasActive    bool
}

func (uc *CheckPromotionsUseCase) Execute(clientID string) (*PromotionInfo, error) {
    // Get client from repository
    client, err := uc.clientRepo.GetByID(clientID)
    if err != nil {
        return nil, err
    }

    // Check active promotion from external service
    hasActive, err := uc.promotionSvc.HasActivePromotion(clientID)
    if err != nil {
        return nil, err
    }

    // Get all promotions from external service
    promotions, err := uc.promotionSvc.GetClientPromotions(clientID)
    if err != nil {
        return nil, err
    }

    return &PromotionInfo{
        ClientName: client.Name,
        Promotions: promotions,
        HasActive:  hasActive,
    }, nil
}
```

### Use Case: Get Client Books

```go
// application/usecases/get_client_books.go
package usecases

import (
    "ExampleProject/domain/entities"
    "ExampleProject/domain/repositories"
)

type GetClientBooksUseCase struct {
    clientRepo repositories.ClientRepository
    bookRepo   repositories.BookRepository
}

func NewGetClientBooksUseCase(
    clientRepo repositories.ClientRepository,
    bookRepo repositories.BookRepository,
) *GetClientBooksUseCase {
    return &GetClientBooksUseCase{
        clientRepo: clientRepo,
        bookRepo:   bookRepo,
    }
}

func (uc *GetClientBooksUseCase) Execute(clientID string) ([]*entities.Book, error) {
    // Get client
    client, err := uc.clientRepo.GetByID(clientID)
    if err != nil {
        return nil, err
    }

    // Get all books borrowed by client
    var books []*entities.Book
    for _, bookID := range client.BorrowedBooks {
        book, err := uc.bookRepo.GetByID(bookID)
        if err != nil {
            continue // Skip books that can't be found
        }
        books = append(books, book)
    }

    return books, nil
}
```

### Use Case: Consume Points

```go
// application/usecases/consume_points.go
package usecases

import (
    "ExampleProject/domain/repositories"
    "ExampleProject/application/services"
    "errors"
)

type ConsumePointsUseCase struct {
    clientRepo    repositories.ClientRepository
    pointsService services.PointsService
}

func NewConsumePointsUseCase(
    clientRepo repositories.ClientRepository,
    pointsService services.PointsService,
) *ConsumePointsUseCase {
    return &ConsumePointsUseCase{
        clientRepo:    clientRepo,
        pointsService: pointsService,
    }
}

func (uc *ConsumePointsUseCase) Execute(clientID string, points int) error {
    // Get client
    client, err := uc.clientRepo.GetByID(clientID)
    if err != nil {
        return err
    }

    // Validate business rule
    if !client.HasMinimumPoints(points) {
        return errors.New("insufficient points")
    }

    // Consume points in external service
    err = uc.pointsService.ConsumePoints(clientID, points)
    if err != nil {
        return err
    }

    // Update local client points
    client.Points -= points

    return uc.clientRepo.Update(client)
}
```

---

## 🔧 Wiring It All Together (Dependency Injection)

```go
// cmd/api/main.go
package main

import (
    "database/sql"
    "log"
    "net/http"
    "os"

    "ExampleProject/application/usecases"
    "ExampleProject/infrastructure/postgresql"
    pgRepos "ExampleProject/infrastructure/postgresql/repositories"
    "ExampleProject/infrastructure/loyalty_points"
    "ExampleProject/infrastructure/promotion"
    "ExampleProject/interfaces/api/handlers"
)

func main() {
    // 1. Initialize database connection (Dependency for repositories)
    db, err := postgresql.NewConnection(postgresql.Config{
        Host:     os.Getenv("DB_HOST"),
        Port:     5432,
        User:     os.Getenv("DB_USER"),
        Password: os.Getenv("DB_PASSWORD"),
        Database: os.Getenv("DB_NAME"),
        SSLMode:  "disable",
    })
    if err != nil {
        log.Fatal("Failed to connect to database:", err)
    }
    defer db.Close()

    // 2. Initialize repositories (Infrastructure layer - DI: inject db)
    // These return domain interfaces, not concrete types
    clientRepo := pgRepos.NewPostgreSQLClientRepository(db)
    bookRepo := pgRepos.NewPostgreSQLBookRepository(db)

    // 3. Initialize external services (Infrastructure layer - DI: inject config)
    // These return application service interfaces
    promotionSvc := promotion.NewPromotionService(
        os.Getenv("PROMOTION_SERVICE_URL"),
        os.Getenv("PROMOTION_API_KEY"),
    )

    pointsSvc := loyalty_points.NewLoyaltyPointsService(
        os.Getenv("LOYALTY_SERVICE_URL"),
        os.Getenv("LOYALTY_API_KEY"),
    )

    // 4. Initialize use cases (Application layer - DI: inject repos & services)
    rentBookUseCase := usecases.NewRentBookUseCase(
        clientRepo,    // domain.repositories.ClientRepository interface
        bookRepo,      // domain.repositories.BookRepository interface
        promotionSvc,  // application.services.PromotionService interface
        pointsSvc,     // application.services.PointsService interface
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

    // 5. Initialize handlers (Interface layer - DI: inject use cases)
    bookHandler := handlers.NewBookHandler(rentBookUseCase)
    clientHandler := handlers.NewClientHandler(
        checkPromotionsUseCase,
        getClientBooksUseCase,
        consumePointsUseCase,
    )

    // 6. Setup routes
    http.HandleFunc("/api/books/rent", bookHandler.RentBook)
    http.HandleFunc("/api/clients/promotions", clientHandler.GetPromotions)
    http.HandleFunc("/api/clients/books", clientHandler.GetBooks)
    http.HandleFunc("/api/clients/points/consume", clientHandler.ConsumePoints)

    // 7. Start server
    log.Println("Server starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### Dependency Injection Flow

```
┌─────────────────────────────────────────────────────────────┐
│                         main.go                             │
│                    (Composition Root)                       │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
   ┌─────────┐         ┌──────────┐         ┌──────────┐
   │   DB    │         │  HTTP    │         │  Config  │
   │ (*sql.DB)│        │  Clients │         │  (env)   │
   └─────────┘         └──────────┘         └──────────┘
        │                     │                     │
        │ inject             │ inject              │ inject
        ▼                     ▼                     ▼
   ┌─────────────────────────────────────────────────────┐
   │         Infrastructure Layer                        │
   │  ┌─────────────┐  ┌─────────────┐  ┌────────────┐ │
   │  │ PostgreSQL  │  │  Loyalty    │  │ Promotion  │ │
   │  │ Repositories│  │  Service    │  │  Service   │ │
   │  └─────────────┘  └─────────────┘  └────────────┘ │
   └─────────────────────────────────────────────────────┘
                              │
                    returns interfaces
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
   repositories.          services.            services.
   ClientRepository      PromotionService     PointsService
   (interface)           (interface)          (interface)
        │                     │                     │
        │ inject              │ inject              │ inject
        ▼                     ▼                     ▼
   ┌─────────────────────────────────────────────────────┐
   │            Application Layer                        │
   │  ┌─────────────┐  ┌─────────────┐  ┌────────────┐ │
   │  │  RentBook   │  │    Check    │  │  Consume   │ │
   │  │  UseCase    │  │ Promotions  │  │   Points   │ │
   │  └─────────────┘  └─────────────┘  └────────────┘ │
   └─────────────────────────────────────────────────────┘
                              │
                    inject use cases
                              │
                              ▼
   ┌─────────────────────────────────────────────────────┐
   │            Interface Layer                          │
   │  ┌─────────────┐  ┌─────────────┐                  │
   │  │    Book     │  │   Client    │                  │
   │  │   Handler   │  │   Handler   │                  │
   │  └─────────────┘  └─────────────┘                  │
   └─────────────────────────────────────────────────────┘
                              │
                    register with router
                              │
                              ▼
                        HTTP Server
```

### Key Dependency Injection Principles

1. **Constructor Injection**: All dependencies passed through constructors
2. **Interface Types**: Functions return interfaces, not concrete types
3. **Composition Root**: All wiring happens in `main.go`
4. **No Global State**: No singletons or global variables
5. **Configuration**: Environment variables injected as needed

---

# Example: Adding a New Feature (CheckCXDoubt)

This section demonstrates how to add a new feature to the project using clean architecture principles. We'll add a service to check if a client (CX) is in doubt with banks, using different providers (Serasa for Brazil, Clearing for Uruguay).

> **⚠️ Important Architectural Note:** The country routing logic (deciding which provider to use based on country) is **business logic**, not infrastructure concern. Therefore, the router that switches between providers belongs in the **application layer**, not in infrastructure. Infrastructure should only contain adapters to external systems (Serasa client, Clearing client), while the application layer orchestrates which adapter to use based on business rules.

---

## Step 1: Define the Service Interface (application/services)

Create the contract in `application/services/README.md`:

```go
// check_cx_doubt_service.go
package services

import "context"

type CheckCXDoubtService interface {
    IsClientInDoubt(ctx context.Context, clientID string, country string) (bool, error)
}
```

---

## Step 2: Define Provider-Specific Interfaces (application/services)

Create specific interfaces for each provider:

```go
// application/services/serasa_service.go
package services

import "context"

type SerasaService interface {
    CheckClientDoubt(ctx context.Context, clientID string) (bool, error)
}

// application/services/clearing_service.go
package services

import "context"

type ClearingService interface {
    CheckClientDoubt(ctx context.Context, clientID string) (bool, error)
}
```

---

## Step 3: Implement Providers in Infrastructure

Create provider-specific implementations:

```go
// infrastructure/serasa/serasa_service_impl.go
package serasa

import (
    "context"
    "net/http"
    // ...
)

type SerasaServiceImpl struct {
    apiURL string
    apiKey string
    client *http.Client
}

func NewSerasaService(apiURL, apiKey string) *SerasaServiceImpl {
    return &SerasaServiceImpl{
        apiURL: apiURL,
        apiKey: apiKey,
        client: &http.Client{},
    }
}

func (s *SerasaServiceImpl) CheckClientDoubt(ctx context.Context, clientID string) (bool, error) {
    // Call Serasa API for Brazil
    // Implementation details: HTTP call, JSON parsing, etc.
    return false, nil
}
```

```go
// infrastructure/clearing/clearing_service_impl.go
package clearing

import (
    "context"
    "net/http"
    // ...
)

type ClearingServiceImpl struct {
    apiURL string
    apiKey string
    client *http.Client
}

func NewClearingService(apiURL, apiKey string) *ClearingServiceImpl {
    return &ClearingServiceImpl{
        apiURL: apiURL,
        apiKey: apiKey,
        client: &http.Client{},
    }
}

func (c *ClearingServiceImpl) CheckClientDoubt(ctx context.Context, clientID string) (bool, error) {
    // Call Clearing API for Uruguay
    // Implementation details: HTTP call, JSON parsing, etc.
    return false, nil
}
```

---

## Step 4: Implement Country Router in Application Layer

The routing logic is **business logic** and belongs in the application layer:

```go
// application/services/check_cx_doubt_router.go
package services

import (
    "context"
    "errors"
)

// CheckCXDoubtRouter handles country-specific routing
type CheckCXDoubtRouter struct {
    serasaService   SerasaService
    clearingService ClearingService
}

func NewCheckCXDoubtRouter(serasa SerasaService, clearing ClearingService) CheckCXDoubtService {
    return &CheckCXDoubtRouter{
        serasaService:   serasa,
        clearingService: clearing,
    }
}

func (r *CheckCXDoubtRouter) IsClientInDoubt(ctx context.Context, clientID string, country string) (bool, error) {
    switch country {
    case "BR", "Brasil", "Brazil":
        return r.serasaService.CheckClientDoubt(ctx, clientID)
    case "UY", "Uruguay":
        return r.clearingService.CheckClientDoubt(ctx, clientID)
    default:
        return false, errors.New("unsupported country: " + country)
    }
}
```

**Why in Application Layer?**
- ✅ Country routing is **business logic** (which provider for which country)
- ✅ Easy to test independently with mocks
- ✅ Can be reused across multiple use cases
- ✅ Doesn't violate dependency rule

---

## Step 5: Wire Everything in main.go

In your composition root, inject the correct implementations:

```go
// main.go
import (
    "ExampleProject/application/services"
    "ExampleProject/infrastructure/serasa"
    "ExampleProject/infrastructure/clearing"
)

func main() {
    // Initialize infrastructure services (adapters to external systems)
    serasaSvc := serasa.NewSerasaService(
        os.Getenv("SERASA_API_URL"),
        os.Getenv("SERASA_API_KEY"),
    )
    
    clearingSvc := clearing.NewClearingService(
        os.Getenv("CLEARING_API_URL"),
        os.Getenv("CLEARING_API_KEY"),
    )
    
    // Create application service with routing logic
    checkCXDoubtSvc := services.NewCheckCXDoubtRouter(serasaSvc, clearingSvc)
    
    // Inject into use case
    rentBookUseCase := usecases.NewRentBookUseCase(
        clientRepo,
        bookRepo,
        promotionSvc,
        pointsSvc,
        checkCXDoubtSvc, // application.services.CheckCXDoubtService interface
    )
}
```

---

## Step 6: Integrate with Use Case

Add the service to your use case (e.g., RentBookUseCase):

```go
// application/usecases/rent_book.go
package usecases

import (
    "context"
    "ExampleProject/application/services"
)

type RentBookUseCase struct {
    checkCXDoubtService services.CheckCXDoubtService
    // ... other dependencies
}

func (uc *RentBookUseCase) Execute(ctx context.Context, clientID, bookID, country string) error {
    inDoubt, err := uc.checkCXDoubtService.IsClientInDoubt(ctx, clientID, country)
    if err != nil {
        return err
    }
    if inDoubt {
        return errors.New("client is in doubt with banks")
    }
    // ... rest of logic
    return nil
}
```

---

## Step 7: Test the Feature

Mock the service for unit tests:

```go
// application/usecases/rent_book_test.go
package usecases_test

import (
    "context"
    "testing"
    "ExampleProject/application/services"
    "ExampleProject/application/usecases"
)

type MockCheckCXDoubtService struct {
    InDoubt bool
    Err    error
}

func (m *MockCheckCXDoubtService) IsClientInDoubt(ctx context.Context, clientID string, country string) (bool, error) {
    return m.InDoubt, m.Err
}

func TestRentBook_CXInDoubt(t *testing.T) {
    mockSvc := &MockCheckCXDoubtService{InDoubt: true}
    uc := usecases.RentBookUseCase{checkCXDoubtService: mockSvc}
    err := uc.Execute(context.Background(), "client1", "book1", "BR")
    if err == nil || err.Error() != "client is in doubt with banks" {
        t.Errorf("expected error for client in doubt, got %v", err)
    }
}
```

---

## Summary

- Define the service interface in application/services
- Implement providers in infrastructure
- Create router in application layer (business logic!)
- Inject into use case via main.go
- Test with mocks

**This pattern keeps your business logic clean, testable, and flexible!**

---

## ✅ Benefits of This Architecture

1. **Testability:** Each layer can be tested independently with mocks
2. **Maintainability:** Clear separation of concerns
3. **Flexibility:** Easy to swap implementations (DB, HTTP framework, etc.)
4. **Business Logic Protection:** Core logic is isolated from external changes
5. **Team Scalability:** Different teams can work on different layers

---

## 🧪 Testing Example

```go
// application/usecases/rent_book_test.go
package usecases_test

import (
    "testing"
    "ExampleProject/domain/entities"
    "ExampleProject/application/usecases"
)

// Mock repository
type MockClientRepository struct {
    client *entities.Client
}

func (m *MockClientRepository) GetByID(id string) (*entities.Client, error) {
    return m.client, nil
}

func (m *MockClientRepository) Update(client *entities.Client) error {
    m.client = client
    return nil
}

// Mock service
type MockPromotionService struct {
    hasPromotion bool
}

func (m *MockPromotionService) HasActivePromotion(clientID string) (bool, error) {
    return m.hasPromotion, nil
}

func TestRentBook_Success(t *testing.T) {
    // Arrange
    client := &entities.Client{
        ID:              "1",
        Name:            "John Doe",
        BorrowedBooks:   []string{},
        MaxBooksAllowed: 3,
        Points:          10,
    }
    
    mockClientRepo := &MockClientRepository{client: client}
    mockPromotionSvc := &MockPromotionService{hasPromotion: true}
    
    useCase := usecases.NewRentBookUseCase(
        mockClientRepo,
        mockBookRepo,
        mockPromotionSvc,
        mockPointsSvc,
    )
    
    // Act
    err := useCase.Execute("1", "book-123")
    
    // Assert
    if err != nil {
        t.Errorf("Expected no error, got %v", err)
    }
    
    if len(client.BorrowedBooks) != 1 {
        t.Errorf("Expected 1 borrowed book, got %d", len(client.BorrowedBooks))
    }
}
```

---

## 📚 Additional Resources

- **Clean Architecture Book** by Robert C. Martin
- **Domain-Driven Design** by Eric Evans
- **Hexagonal Architecture** (Ports and Adapters)

---

## 🎯 Quick Reference

### Where to put new code?

| What you're adding | Where it goes |
|-------------------|---------------|
| Business rules for an entity | `domain/entities/` |
| New data access method | Interface in `domain/repositories/`, implementation in `infrastructure/repositories/` |
| External service integration | Interface in `application/services/`, implementation in `infrastructure/services/` |
| Application logic/workflow | `application/usecases/` |
| HTTP endpoint | `interfaces/api/handlers/` |
| Request/Response structure | `interfaces/dto/` |
| Database code | `infrastructure/repositories/` |
| Third-party API calls | `infrastructure/services/` or `infrastructure/http/` |

---

## 🔍 Key Principles to Remember

1. **Dependency Rule:** Source code dependencies must point only inward
2. **Interface Segregation:** Define interfaces where they're used, implement where convenient
3. **Dependency Injection:** Pass dependencies through constructors
4. **Single Responsibility:** Each layer has one reason to change
5. **Business Rules First:** Start with domain, work outward

---

This structure ensures your codebase remains clean, testable, and maintainable as it grows!
