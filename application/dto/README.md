# Application DTOs

## Purpose

Application DTOs define the **data contracts** that the application layer expects. These are used by:
- Use cases
- Repository interfaces
- Service interfaces

**Key Principle**: The application defines what it expects through these DTOs. Infrastructure adapts external formats (database records, API responses) to these DTOs.

## Why Application DTOs?

### Separation of Concerns
- **Application DTOs**: What the application needs
- **Infrastructure Entities**: How external systems represent data
- **API DTOs** (`interfaces/dto/`): What the API exposes to clients

### Benefits
- Application is independent of infrastructure details
- Can swap databases without changing application logic
- Infrastructure changes don't affect application layer
- Clear contracts between layers

## Rules

✅ **DO:**
- Define clear data structures
- Use simple Go types
- Include only data, no behavior
- Document fields clearly
- Keep DTOs focused and cohesive

❌ **DON'T:**
- Include business logic
- Depend on external libraries
- Use database-specific types (sql.NullString, etc.)
- Use HTTP-specific types
- Add framework dependencies

## Example: Client DTO

```go
// client_dto.go
package dto

// ClientDTO represents a client in the application layer
// Used by use cases, repositories, and services
type ClientDTO struct {
    ID              string
    Name            string
    Email           string
    Points          int
    MaxBooksAllowed int
    BorrowedBookIDs []string
}
```

## Example: Book DTO

```go
// book_dto.go
package dto

import "time"

// BookDTO represents a book in the application layer
type BookDTO struct {
    ID          string
    Title       string
    Author      string
    ISBN        string
    IsAvailable bool
    BorrowedBy  string
    DueDate     *time.Time
}
```

## Example: Points DTO

```go
// points_dto.go
package dto

// PointsDTO represents loyalty points information
// Returned by PointsService
type PointsDTO struct {
    ClientID        string
    Points          int
    Tier            string
    HasDiscount     bool
    DiscountPercent int
}

// PointsTransactionDTO represents a points transaction
type PointsTransactionDTO struct {
    ID        string
    ClientID  string
    Points    int
    Type      string // "earn", "spend"
    Reason    string
    CreatedAt time.Time
}
```

## Example: Promotion DTO

```go
// promotion_dto.go
package dto

import "time"

// PromotionDTO represents a promotion
type PromotionDTO struct {
    ID              string
    Code            string
    Description     string
    DiscountPercent int
    Active          bool
    ValidFrom       time.Time
    ValidUntil      time.Time
}

// PromotionResultDTO represents the result of checking promotions
type PromotionResultDTO struct {
    HasPromotion    bool
    Promotions      []*PromotionDTO
    TotalDiscount   int
}
```

## Flow Example

### 1. Application Layer Defines DTO
```go
// application/dto/client_dto.go
type ClientDTO struct {
    ID    string
    Name  string
    Email string
}
```

### 2. Application Layer Uses DTO in Interface
```go
// application/repositories/client_repository.go
type ClientRepository interface {
    GetByID(id string) (*dto.ClientDTO, error)
    Save(client *dto.ClientDTO) error
}
```

### 3. Infrastructure Adapts to DTO
```go
// infrastructure/postgresql/entities/client_record.go
type ClientRecord struct {
    ID        string
    Name      string
    Email     string
    CreatedAt time.Time  // Database specific
    UpdatedAt time.Time  // Database specific
}

func (r *ClientRecord) ToDTO() *dto.ClientDTO {
    return &dto.ClientDTO{
        ID:    r.ID,
        Name:  r.Name,
        Email: r.Email,
    }
}
```

### 4. Infrastructure Repository Implementation
```go
// infrastructure/postgresql/repositories/client_repository.go
func (r *PostgreSQLClientRepository) GetByID(id string) (*dto.ClientDTO, error) {
    var record ClientRecord
    // ... query database ...
    
    // Convert infrastructure entity to application DTO
    return record.ToDTO(), nil
}
```

## Comparison: Application DTO vs API DTO

### Application DTO (`application/dto/`)
```go
// What the application layer needs
type ClientDTO struct {
    ID              string
    Name            string
    Email           string
    BorrowedBookIDs []string
}
```

### API DTO (`interfaces/dto/`)
```go
// What the API exposes to external clients
type ClientResponse struct {
    ID       string              `json:"id"`
    Name     string              `json:"name"`
    Email    string              `json:"email"`
    Books    []BookSummary       `json:"borrowed_books"`
    Links    map[string]string   `json:"_links"`
}
```

The API layer converts between Application DTOs and API DTOs.

---

**Remember**: Application DTOs are the contracts that define what your application expects. Infrastructure and interface layers adapt to these contracts.

