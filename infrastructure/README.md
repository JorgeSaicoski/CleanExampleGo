# Infrastructure Layer

## Purpose

The infrastructure layer contains **concrete implementations** of interfaces defined in the **application layer**. This is where you interact with:
- Databases (PostgreSQL, MySQL, MongoDB, etc.)
- External APIs and microservices
- File systems
- Message queues
- Cache systems
- Third-party services

## Key Architectural Principle: Application Defines, Infrastructure Adapts

### The Dependency Rule

**Application Layer** (high-level) defines:
- Repository **interfaces** (`application/repositories/`)
- Service **interfaces** (`application/services/`)
- **DTOs** that describe what it expects (`application/dto/`)
- Business use cases

**Infrastructure Layer** (low-level) implements:
- Application interfaces using concrete technologies (PostgreSQL, HTTP APIs, etc.)
- **Internal entities** for external system mapping (`postgresql/entities/`, `loyalty_points/entities/`)
- Conversion from external formats to application DTOs

### Why This Matters

```
Application says: "I need ClientRepository.GetByID(userID) returning ClientDTO"
Infrastructure says: "I'll query PostgreSQL and convert the result to ClientDTO"

Application says: "I need PointsService.GetBalance(clientID) returning PointsDTO"
Infrastructure says: "I'll call the Loyalty API and convert the response to PointsDTO"
```

This follows the **Dependency Inversion Principle**:
- High-level modules (application) don't depend on low-level modules (infrastructure)
- Both depend on abstractions (interfaces defined in application layer)
- The application doesn't know about PostgreSQL or HTTP APIs, only about its interfaces and DTOs

### Benefits

- **Flexibility**: Swap PostgreSQL for MongoDB without changing application code
- **Testability**: Mock interfaces for unit tests
- **Maintainability**: Infrastructure changes don't affect business logic
- **Multiple Implementations**: Support multiple databases or APIs simultaneously

## Folder Structure

```
infrastructure/
├── postgresql/              ← PostgreSQL database implementation
│   ├── entities/           ← Database-specific entity mappings (internal)
│   ├── repositories/       ← Implements application/repositories interfaces
│   └── migrations/         ← Database migration files
│
├── loyalty_points/         ← Loyalty points system (external service)
│   ├── entities/           ← Loyalty-specific entities (internal)
│   ├── client.go           ← HTTP client for loyalty service
│   └── service_impl.go     ← Implements application/services interface
│
├── promotion/              ← Promotion system (external service)
│   ├── entities/           ← Promotion-specific entities (internal)
│   ├── client.go           ← HTTP client for promotion service
│   └── service_impl.go     ← Implements application/services interface
│
├── notification/           ← Notification system
│   ├── email/              ← Email service implementation
│   ├── sms/                ← SMS service implementation
│   └── push/               ← Push notification implementation
│
└── http/                   ← Shared HTTP utilities
    ├── client.go           ← Reusable HTTP client
    ├── retry.go            ← Retry logic
    └── circuit_breaker.go  ← Circuit breaker pattern
```

**Note**: DTOs and interfaces are defined in the `application/` layer.
Infrastructure adapts external responses to application DTOs.

## Key Principles

### 1. Dependency Injection

Each infrastructure component should:
- Accept configuration via constructor
- Implement interfaces from the application layer
- Be easily mockable for testing

**Example:**
```go
// Application defines the interface and DTO
// application/repositories/client_repository.go
type ClientRepository interface {
    GetByID(id string) (*dto.ClientDTO, error)
}

// Infrastructure implements it
// infrastructure/postgresql/repositories/client_repository.go
type PostgreSQLClientRepository struct {
    db *sql.DB
}

func NewPostgreSQLClientRepository(db *sql.DB) repositories.ClientRepository {
    return &PostgreSQLClientRepository{db: db}
}
```

### 2. Internal Entities for External Systems

Each external system should have its own internal entities to isolate external format changes.
These entities convert to **application DTOs**:

```go
// loyalty_points/entities/points_response.go (internal to infrastructure)
type PointsBalanceResponse struct {
    ClientID string `json:"client_id"`
    Balance  int    `json:"balance"`
    Tier     string `json:"tier"`
}

// Convert to application DTO
func (r *PointsBalanceResponse) ToDTO() *dto.PointsDTO {
    return &dto.PointsDTO{
        ClientID: r.ClientID,
        Points:   r.Balance,
        Tier:     r.Tier,
    }
}
```

### 3. Entities Map to External Systems

Infrastructure entities are **internal** and map to external systems (databases, APIs).
They convert to/from **application DTOs**:

```go
// postgresql/entities/client_record.go (internal to infrastructure)
type ClientRecord struct {
    ID              string
    Name            string
    Email           string
    Points          int
    MaxBooksAllowed int
    CreatedAt       time.Time
    UpdatedAt       time.Time
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

// Convert from application DTO
func FromClientDTO(client *dto.ClientDTO) *ClientRecord {
    return &ClientRecord{
        ID:              client.ID,
        Name:            client.Name,
        Email:           client.Email,
        Points:          client.Points,
        MaxBooksAllowed: client.MaxBooksAllowed,
    }
}
```

## Example: PostgreSQL Implementation

### Structure
```
postgresql/
├── entities/                ← Database records (internal)
│   ├── client_record.go
│   └── book_record.go
├── repositories/            ← Implements application/repositories
│   ├── client_repository.go
│   └── book_repository.go
├── migrations/              ← SQL migrations
└── connection.go            ← DB connection setup
```

### Implementation
```go
// postgresql/repositories/client_repository.go
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

func NewPostgreSQLClientRepository(db *sql.DB) repositories.ClientRepository {
    return &PostgreSQLClientRepository{db: db}
}

// Implements application interface, returns application DTO
func (r *PostgreSQLClientRepository) GetByID(id string) (*dto.ClientDTO, error) {
    query := `SELECT id, name, email, points, max_books_allowed FROM clients WHERE id = $1`
    
    var record pgEntities.ClientRecord
    err := r.db.QueryRow(query, id).Scan(
        &record.ID,
        &record.Name,
        &record.Email,
        &record.Points,
        &record.MaxBooksAllowed,
    )
    
    if err != nil {
        return nil, err
    }
    
    // Convert infrastructure entity to application DTO
    return record.ToDTO(), nil
}
```

## Example: Loyalty Points System

### Structure
```
loyalty_points/
├── entities/                  ← Internal entities for API responses
│   ├── points_balance.go      ← Loyalty API response structure
│   └── transaction.go         ← Points transaction structure
├── client.go                  ← HTTP client wrapper
└── service_impl.go            ← Implements application/services interface
```

### Entities (Internal to Infrastructure)
```go
// loyalty_points/entities/points_balance.go
package entities

import "ExampleProject/application/dto"

// Internal entity representing loyalty API response
type PointsBalance struct {
    ClientID        string `json:"client_id"`
    TotalPoints     int    `json:"total_points"`
    AvailablePoints int    `json:"available_points"`
    Tier            string `json:"tier"`
    HasDiscount     bool   `json:"has_discount"`
    DiscountPercent int    `json:"discount_percent"`
}

// Convert to application DTO
func (p *PointsBalance) ToDTO() *dto.PointsDTO {
    return &dto.PointsDTO{
        ClientID:        p.ClientID,
        Points:          p.AvailablePoints,
        Tier:            p.Tier,
        HasDiscount:     p.HasDiscount,
        DiscountPercent: p.DiscountPercent,
    }
}

// loyalty_points/entities/transaction.go
type PointsTransaction struct {
    ID        string    `json:"id"`
    ClientID  string    `json:"client_id"`
    Points    int       `json:"points"`
    Type      string    `json:"type"` // "earn", "spend"
    Reason    string    `json:"reason"`
    CreatedAt time.Time `json:"created_at"`
}
```

### Service Implementation
```go
// loyalty_points/service_impl.go
package loyalty_points

import (
    "ExampleProject/application/dto"
    "ExampleProject/application/services"
    "ExampleProject/infrastructure/loyalty_points/entities"
)

type LoyaltyPointsService struct {
    client *HTTPClient
    baseURL string
}

func NewLoyaltyPointsService(baseURL string) services.PointsService {
    return &LoyaltyPointsService{
        client: NewHTTPClient(),
        baseURL: baseURL,
    }
}

// Implements application interface, returns application DTO
func (s *LoyaltyPointsService) GetBalance(clientID string) (*dto.PointsDTO, error) {
    var response entities.PointsBalance
    
    err := s.client.Get(
        s.baseURL + "/api/points/" + clientID,
        &response,
    )
    
    if err != nil {
        return nil, err
    }
    
    // Convert infrastructure entity to application DTO
    return response.ToDTO(), nil
}
```

type LoyaltyPointsService struct {
    client *LoyaltyClient
}

func NewLoyaltyPointsService(baseURL, apiKey string) services.PointsService {
    return &LoyaltyPointsService{
        client: NewLoyaltyClient(baseURL, apiKey),
    }
}

func (s *LoyaltyPointsService) GetClientPoints(clientID string) (int, error) {
    balance, err := s.client.GetBalance(clientID)
    if err != nil {
        return 0, err
    }
    return balance.AvailablePoints, nil
}

func (s *LoyaltyPointsService) CheckDiscount(clientID string) (bool, int, error) {
    req := &dto.CheckDiscountRequest{
        ClientID: clientID,
    }
    
    resp, err := s.client.CheckDiscount(req)
    if err != nil {
        return false, 0, err
    }
    
    return resp.HasDiscount, resp.DiscountPercent, nil
}

func (s *LoyaltyPointsService) AddPoints(clientID string, points int) error {
    return s.client.AddPoints(clientID, points, "book_rental")
}

func (s *LoyaltyPointsService) ConsumePoints(clientID string, points int) error {
    return s.client.ConsumePoints(clientID, points)
}
```

### HTTP Client
```go
// loyalty_points/client.go
package loyalty_points

import (
    "bytes"
    "encoding/json"
    "fmt"
    "net/http"
    "ExampleProject/infrastructure/loyalty_points/dto"
    "ExampleProject/infrastructure/loyalty_points/entities"
)

type LoyaltyClient struct {
    baseURL string
    apiKey  string
    client  *http.Client
}

func NewLoyaltyClient(baseURL, apiKey string) *LoyaltyClient {
    return &LoyaltyClient{
        baseURL: baseURL,
        apiKey:  apiKey,
        client:  &http.Client{},
    }
}

func (c *LoyaltyClient) GetBalance(clientID string) (*entities.PointsBalance, error) {
    url := fmt.Sprintf("%s/api/v1/points/balance/%s", c.baseURL, clientID)
    
    req, _ := http.NewRequest("GET", url, nil)
    c.addAuthHeaders(req)
    
    resp, err := c.client.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
    
    var dtoResp dto.PointsBalanceResponse
    if err := json.NewDecoder(resp.Body).Decode(&dtoResp); err != nil {
        return nil, err
    }
    
    return dtoResp.ToEntity(), nil
}

func (c *LoyaltyClient) CheckDiscount(req *dto.CheckDiscountRequest) (*dto.CheckDiscountResponse, error) {
    url := fmt.Sprintf("%s/api/v1/discounts/check", c.baseURL)
    
    jsonData, _ := json.Marshal(req)
    httpReq, _ := http.NewRequest("POST", url, bytes.NewBuffer(jsonData))
    c.addAuthHeaders(httpReq)
    httpReq.Header.Set("Content-Type", "application/json")
    
    resp, err := c.client.Do(httpReq)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
    
    var dtoResp dto.CheckDiscountResponse
    if err := json.NewDecoder(resp.Body).Decode(&dtoResp); err != nil {
        return nil, err
    }
    
    return &dtoResp, nil
}

func (c *LoyaltyClient) addAuthHeaders(req *http.Request) {
    req.Header.Set("Authorization", "Bearer "+c.apiKey)
    req.Header.Set("Accept", "application/json")
}
```

## Wiring It Together (Dependency Injection)

```go
// cmd/api/main.go
func main() {
    // 1. Initialize infrastructure dependencies
    
    // Database connection (injected into repositories)
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
    
    // 2. Create repository implementations (DI: inject *sql.DB)
    clientRepo := postgresql.NewPostgreSQLClientRepository(db)
    bookRepo := postgresql.NewPostgreSQLBookRepository(db)
    
    // 3. Create external service implementations (DI: inject configuration)
    loyaltyService := loyalty_points.NewLoyaltyPointsService(
        os.Getenv("LOYALTY_SERVICE_URL"),
        os.Getenv("LOYALTY_API_KEY"),
    )
    
    promotionService := promotion.NewPromotionService(
        os.Getenv("PROMOTION_SERVICE_URL"),
        os.Getenv("PROMOTION_API_KEY"),
    )
    
    notificationService := notification.NewEmailService(
        os.Getenv("SMTP_HOST"),
        587,
        os.Getenv("SMTP_USER"),
        os.Getenv("SMTP_PASSWORD"),
    )
    
    // 4. Create use cases (DI: inject repositories and services)
    rentBookUseCase := usecases.NewRentBookUseCase(
        clientRepo,        // Interface: repositories.ClientRepository
        bookRepo,          // Interface: repositories.BookRepository
        promotionService,  // Interface: services.PromotionService
        loyaltyService,    // Interface: services.PointsService
    )
    
    returnBookUseCase := usecases.NewReturnBookUseCase(
        clientRepo,
        bookRepo,
        loyaltyService,
        notificationService,
    )
    
    // 5. Create handlers (DI: inject use cases)
    bookHandler := handlers.NewBookHandler(
        rentBookUseCase,
        returnBookUseCase,
    )
    
    // 6. Setup HTTP server
    router := setupRouter(bookHandler)
    server := &http.Server{
        Addr:    ":8080",
        Handler: router,
    }
    
    log.Println("Server starting on :8080")
    log.Fatal(server.ListenAndServe())
}
```

### Dependency Injection Flow

```
main.go
  │
  ├─> Creates DB connection
  │     └─> Injects into PostgreSQL repositories
  │
  ├─> Creates HTTP config
  │     └─> Injects into external service clients
  │
  ├─> Creates repositories (return interface types)
  │     └─> Injects into use cases
  │
  ├─> Creates services (return interface types)
  │     └─> Injects into use cases
  │
  ├─> Creates use cases
  │     └─> Injects into handlers
  │
  └─> Creates handlers
        └─> Registers with HTTP router
```

**Key Principles:**
1. Dependencies flow from outer to inner layers
2. Constructors receive dependencies (DI)
3. Constructors return interface types
4. Configuration comes from environment variables
5. No global state or singletons

## Best Practices

1. **Isolation**: Each external system has its own folder with entities and DTOs
2. **Conversion**: Always convert between infrastructure entities and domain entities
3. **Configuration**: Accept configuration via constructors
4. **Error Handling**: Convert infrastructure errors to domain errors
5. **Testing**: Easy to mock because of dependency injection
6. **Logging**: Log all external calls
7. **Resilience**: Implement retries, timeouts, circuit breakers

## Benefits

- ✅ **Clear boundaries**: Each system is isolated
- ✅ **Easy to swap**: Change database from PostgreSQL to MongoDB without touching domain
- ✅ **Testable**: Mock any infrastructure component
- ✅ **Maintainable**: Changes to external APIs only affect one folder
- ✅ **Type-safe**: Explicit conversions between layers

---

**Implement your infrastructure components following this structure!**

