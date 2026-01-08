# 🔌 Dependency Injection Guide

## What is Dependency Injection?

Dependency Injection (DI) is a design pattern where objects receive their dependencies from external sources rather than creating them internally. This makes code more testable, maintainable, and flexible.

## Project Dependency Injection Flow

```
main.go (Composition Root)
    │
    ├─> Create Infrastructure Dependencies
    │   ├─> Database connections
    │   ├─> HTTP clients
    │   └─> Configuration
    │
    ├─> Create Repositories (inject DB)
    │   └─> Return: domain repository interfaces
    │
    ├─> Create Services (inject config/HTTP clients)
    │   └─> Return: application service interfaces
    │
    ├─> Create Use Cases (inject repositories & services)
    │   └─> Return: use case structs
    │
    └─> Create Handlers (inject use cases)
        └─> Register with HTTP router
```

## Layer-by-Layer Dependency Injection

### 1. Infrastructure Layer

#### PostgreSQL Repositories

**Interface Definition (Domain Layer):**
```go
// domain/repositories/client_repository.go
package repositories

import "ExampleProject/domain/entities"

type ClientRepository interface {
    GetByID(id string) (*entities.Client, error)
    Save(client *entities.Client) error
    Update(client *entities.Client) error
}
```

**Implementation (Infrastructure Layer):**
```go
// infrastructure/postgresql/repositories/client_repository.go
package repositories

import (
    "database/sql"
    "ExampleProject/domain/entities"
    "ExampleProject/domain/repositories"
)

type PostgreSQLClientRepository struct {
    db *sql.DB  // Dependency injected via constructor
}

// Constructor receives dependency and returns interface type
func NewPostgreSQLClientRepository(db *sql.DB) repositories.ClientRepository {
    return &PostgreSQLClientRepository{db: db}
}

func (r *PostgreSQLClientRepository) GetByID(id string) (*entities.Client, error) {
    // Implementation uses injected db
    // ...
}
```

**Why This Works:**
- ✅ Repository receives `*sql.DB` via constructor
- ✅ Returns domain interface, not concrete type
- ✅ Easy to mock for testing
- ✅ Can swap PostgreSQL for MySQL without changing domain

#### External Service Implementations

**Interface Definition (Application Layer):**
```go
// application/services/points_service.go
package services

type PointsService interface {
    GetClientPoints(clientID string) (int, error)
    AddPoints(clientID string, points int) error
    ConsumePoints(clientID string, points int) error
}
```

**Implementation (Infrastructure Layer):**
```go
// infrastructure/loyalty_points/service_impl.go
package loyalty_points

import (
    "ExampleProject/application/services"
    "ExampleProject/infrastructure/http"
)

type LoyaltyPointsService struct {
    client *LoyaltyClient  // Internal dependency
}

// Constructor receives configuration and returns interface type
func NewLoyaltyPointsService(baseURL, apiKey string) services.PointsService {
    return &LoyaltyPointsService{
        client: NewLoyaltyClient(baseURL, apiKey),
    }
}

func (s *LoyaltyPointsService) GetClientPoints(clientID string) (int, error) {
    // Implementation uses injected client
    // ...
}
```

**Why This Works:**
- ✅ Service receives configuration via constructor
- ✅ Returns application interface, not concrete type
- ✅ Easy to mock for testing
- ✅ Can swap loyalty system without changing use cases

### 2. Application Layer (Use Cases)

**Use Case with Multiple Dependencies:**
```go
// application/usecases/rent_book.go
package usecases

import (
    "ExampleProject/domain/repositories"
    "ExampleProject/application/services"
)

type RentBookUseCase struct {
    clientRepo   repositories.ClientRepository  // Domain interface
    bookRepo     repositories.BookRepository    // Domain interface
    promotionSvc services.PromotionService      // Application interface
    pointsSvc    services.PointsService         // Application interface
}

// Constructor receives all dependencies (DI)
func NewRentBookUseCase(
    clientRepo repositories.ClientRepository,
    bookRepo repositories.BookRepository,
    promotionSvc services.PromotionService,
    pointsSvc services.PointsService,
) *RentBookUseCase {
    return &RentBookUseCase{
        clientRepo:   clientRepo,
        bookRepo:     bookRepo,
        promotionSvc: promotionSvc,
        pointsSvc:    pointsSvc,
    }
}

func (uc *RentBookUseCase) Execute(clientID, bookID string) error {
    // Use injected dependencies
    client, err := uc.clientRepo.GetByID(clientID)
    // ...
}
```

**Why This Works:**
- ✅ Use case receives interfaces, not concrete types
- ✅ No knowledge of database or HTTP implementation
- ✅ Easy to test with mocks
- ✅ Follows Dependency Inversion Principle

### 3. Interface Layer (Handlers)

**Handler with Use Case Dependency:**
```go
// interfaces/api/handlers/book_handler.go
package handlers

import (
    "ExampleProject/application/usecases"
    "ExampleProject/interfaces/dto"
)

type BookHandler struct {
    rentBookUseCase *usecases.RentBookUseCase  // Use case dependency
}

// Constructor receives use case (DI)
func NewBookHandler(rentBookUseCase *usecases.RentBookUseCase) *BookHandler {
    return &BookHandler{
        rentBookUseCase: rentBookUseCase,
    }
}

func (h *BookHandler) RentBook(w http.ResponseWriter, r *http.Request) {
    // Parse request
    var req dto.RentBookRequest
    // ...
    
    // Call use case
    err := h.rentBookUseCase.Execute(req.ClientID, req.BookID)
    // ...
}
```

**Why This Works:**
- ✅ Handler receives use case via constructor
- ✅ No knowledge of repositories or services
- ✅ Easy to test
- ✅ Single responsibility: HTTP handling only

### 4. Composition Root (main.go)

**Wiring Everything Together:**
```go
// cmd/api/main.go
package main

func main() {
    // 1. Create primitive dependencies
    db := createDatabase()
    config := loadConfig()
    
    // 2. Create infrastructure (inject primitives)
    clientRepo := postgresql.NewPostgreSQLClientRepository(db)
    bookRepo := postgresql.NewPostgreSQLBookRepository(db)
    loyaltyService := loyalty_points.NewLoyaltyPointsService(
        config.LoyaltyURL,
        config.LoyaltyAPIKey,
    )
    promotionService := promotion.NewPromotionService(
        config.PromotionURL,
        config.PromotionAPIKey,
    )
    
    // 3. Create use cases (inject repositories & services)
    rentBookUseCase := usecases.NewRentBookUseCase(
        clientRepo,      // Injected
        bookRepo,        // Injected
        promotionService,// Injected
        loyaltyService,  // Injected
    )
    
    // 4. Create handlers (inject use cases)
    bookHandler := handlers.NewBookHandler(rentBookUseCase)
    
    // 5. Start server
    startServer(bookHandler)
}
```

## Testing with Dependency Injection

### Mocking Repositories

```go
// Mock implementation
type MockClientRepository struct {
    GetByIDFunc func(id string) (*entities.Client, error)
}

func (m *MockClientRepository) GetByID(id string) (*entities.Client, error) {
    return m.GetByIDFunc(id)
}

// Test
func TestRentBookUseCase_Success(t *testing.T) {
    // Create mock
    mockRepo := &MockClientRepository{
        GetByIDFunc: func(id string) (*entities.Client, error) {
            return &entities.Client{ID: id, Name: "Test"}, nil
        },
    }
    
    // Inject mock into use case
    useCase := usecases.NewRentBookUseCase(
        mockRepo,      // Mock injected
        mockBookRepo,
        mockPromotion,
        mockPoints,
    )
    
    // Test
    err := useCase.Execute("client1", "book1")
    assert.NoError(t, err)
}
```

### Mocking Services

```go
// Mock implementation
type MockPointsService struct {
    AddPointsFunc func(clientID string, points int) error
}

func (m *MockPointsService) AddPoints(clientID string, points int) error {
    if m.AddPointsFunc != nil {
        return m.AddPointsFunc(clientID, points)
    }
    return nil
}

// Test
func TestRentBook_WithPoints(t *testing.T) {
    pointsAdded := 0
    mockPoints := &MockPointsService{
        AddPointsFunc: func(clientID string, points int) error {
            pointsAdded = points
            return nil
        },
    }
    
    useCase := usecases.NewRentBookUseCase(
        mockClientRepo,
        mockBookRepo,
        mockPromotion,
        mockPoints,  // Mock injected
    )
    
    useCase.Execute("client1", "book1")
    assert.Equal(t, 10, pointsAdded)
}
```

## Common Dependency Injection Patterns

### Pattern 1: Constructor Injection (Recommended)

```go
type Service struct {
    dep1 Dependency1
    dep2 Dependency2
}

func NewService(dep1 Dependency1, dep2 Dependency2) *Service {
    return &Service{
        dep1: dep1,
        dep2: dep2,
    }
}
```

### Pattern 2: Interface Return Types

```go
// Return interface, not concrete type
func NewRepository(db *sql.DB) repositories.ClientRepository {
    return &PostgreSQLClientRepository{db: db}
}

// Not this:
func NewRepository(db *sql.DB) *PostgreSQLClientRepository {
    return &PostgreSQLClientRepository{db: db}
}
```

### Pattern 3: Configuration Objects

```go
type ServiceConfig struct {
    BaseURL string
    APIKey  string
    Timeout time.Duration
}

func NewService(config ServiceConfig) Service {
    return &ServiceImpl{
        baseURL: config.BaseURL,
        apiKey:  config.APIKey,
        timeout: config.Timeout,
    }
}
```

## Anti-Patterns to Avoid

### ❌ Global Variables
```go
// DON'T DO THIS
var DB *sql.DB

func init() {
    DB, _ = sql.Open("postgres", "...")
}

func (r *Repository) GetByID(id string) {
    DB.Query(...) // Using global
}
```

### ❌ Singletons
```go
// DON'T DO THIS
type Database struct {
    db *sql.DB
}

var instance *Database

func GetInstance() *Database {
    if instance == nil {
        instance = &Database{}
    }
    return instance
}
```

### ❌ Service Locator
```go
// DON'T DO THIS
func (uc *UseCase) Execute() {
    repo := ServiceLocator.GetRepository() // Bad!
    repo.GetByID("123")
}
```

### ✅ Do This Instead
```go
// Use constructor injection
func NewUseCase(repo Repository) *UseCase {
    return &UseCase{repo: repo}
}
```

## Benefits of Dependency Injection

1. **Testability**: Easy to inject mocks
2. **Flexibility**: Easy to swap implementations
3. **Maintainability**: Clear dependencies
4. **Loose Coupling**: Components don't know about implementations
5. **Single Responsibility**: Each component has one job

## Dependency Injection Checklist

- [ ] All dependencies passed through constructors
- [ ] No global variables
- [ ] No singletons
- [ ] Constructors return interface types
- [ ] All wiring happens in main.go
- [ ] Easy to create mocks for testing
- [ ] Clear dependency flow: outer → inner

---

**This project follows dependency injection throughout - every component receives its dependencies via constructors!**

