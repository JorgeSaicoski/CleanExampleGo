# Application Repositories (Interfaces)

## Purpose

Repository interfaces define **contracts for data access** that the application layer expects. They are defined here and implemented in the infrastructure layer.

**Key Principle**: The application defines what data access it needs. Infrastructure adapts to provide it.

## Rules

✅ **DO:**
- Define interfaces for data access
- Use **application DTOs** as parameters and return types
- Keep interfaces focused (Interface Segregation Principle)
- Use meaningful method names
- Document expected behavior

❌ **DON'T:**
- Implement data access logic here (that goes in infrastructure)
- Use database-specific types (like sql.Rows, BSON, etc.)
- Use infrastructure entities in signatures
- Include HTTP or framework code
- Couple to a specific storage mechanism

## Architecture Flow

```
Use Case (application/usecases)
    ↓ depends on
Repository Interface (application/repositories) ← YOU ARE HERE
    ↑ implements
PostgreSQL Repository (infrastructure/postgresql/repositories)
```

## Example: Client Repository Interface

```go
// client_repository.go
package repositories

import "ExampleProject/application/dto"

// ClientRepository defines the contract for client data access
// Infrastructure layer will implement this interface
type ClientRepository interface {
    // Basic CRUD operations
    GetByID(id string) (*dto.ClientDTO, error)
    GetAll() ([]*dto.ClientDTO, error)
    Save(client *dto.ClientDTO) error
    Update(client *dto.ClientDTO) error
    Delete(id string) error
    
    // Custom query methods
    GetByEmail(email string) (*dto.ClientDTO, error)
    GetClientsWithOverdueBooks() ([]*dto.ClientDTO, error)
    GetTopClientsByPoints(limit int) ([]*dto.ClientDTO, error)
}
```

## Example: Book Repository Interface

```go
// book_repository.go
package repositories

import "ExampleProject/application/dto"

// BookRepository defines the contract for book data access
type BookRepository interface {
    // Basic CRUD operations
    GetByID(id string) (*dto.BookDTO, error)
    GetAll() ([]*dto.BookDTO, error)
    Save(book *dto.BookDTO) error
    Update(book *dto.BookDTO) error
    Delete(id string) error
    
    // Custom query methods
    FindByISBN(isbn string) (*dto.BookDTO, error)
    FindAvailableBooks() ([]*dto.BookDTO, error)
    FindByBorrower(clientID string) ([]*dto.BookDTO, error)
    FindOverdueBooks() ([]*dto.BookDTO, error)
    SearchByTitle(query string) ([]*dto.BookDTO, error)
    SearchByAuthor(author string) ([]*dto.BookDTO, error)
}
```

## Why Application DTOs?

### ✅ Correct: Application DTOs
```go
// Application defines what it needs
type ClientRepository interface {
    GetByID(id string) (*dto.ClientDTO, error)
}
```

### ❌ Incorrect: Domain Entities
```go
// DON'T use domain entities - they may have business logic
type ClientRepository interface {
    GetByID(id string) (*entities.Client, error)
}
```

### ❌ Incorrect: Infrastructure Types
```go
// DON'T use infrastructure types - couples to database
type ClientRepository interface {
    GetByID(id string) (*ClientRecord, error)
}
```

## Method Naming Conventions

### Read Operations
- `Get{Entity}ByID` - Get single entity by ID (returns error if not found)
- `Get{Entity}By{Field}` - Get single entity by field
- `GetAll{Entities}` - Get all entities
- `Find{Entities}By{Criteria}` - Find multiple entities (returns empty slice if none)
- `Search{Entities}` - Search with query string
- `Count{Entities}` - Count entities matching criteria

### Write Operations
- `Save{Entity}` - Create new entity
- `Update{Entity}` - Update existing entity
- `Delete{Entity}` - Delete entity by ID
- `DeleteBy{Field}` - Delete by criteria

### Examples
```go
type UserRepository interface {
    GetUserByID(id string) (*dto.UserDTO, error)
    GetUserByEmail(email string) (*dto.UserDTO, error)
    GetAllUsers() ([]*dto.UserDTO, error)
    FindActiveUsers() ([]*dto.UserDTO, error)
    SearchUsersByName(query string) ([]*dto.UserDTO, error)
    CountUsers() (int, error)
    SaveUser(user *dto.UserDTO) error
    UpdateUser(user *dto.UserDTO) error
    DeleteUser(id string) error
}
```

## Return Types and Error Handling

### Single Entity
```go
GetByID(id string) (*dto.ClientDTO, error)
```
- Return pointer to DTO or `nil` with error if not found
- Return error for actual errors (DB connection, query failures, etc.)

### Multiple Entities
```go
GetAll() ([]*dto.ClientDTO, error)
```
- Return empty slice if no results (not an error)
- Return error only for actual failures

### Example Implementation Pattern
```go
func (r *PostgreSQLClientRepository) GetByID(id string) (*dto.ClientDTO, error) {
    // Query database
    var record ClientRecord
    err := r.db.QueryRow("SELECT ... WHERE id = $1", id).Scan(...)
    
    if err == sql.ErrNoRows {
        return nil, fmt.Errorf("client not found: %s", id)
    }
    if err != nil {
        return nil, fmt.Errorf("database error: %w", err)
    }
    
    // Convert infrastructure entity to application DTO
    return record.ToDTO(), nil
}
```

## Testing with Repository Interfaces

Create mocks for testing use cases:

```go
// mock_client_repository.go
package repositories_test

import "ExampleProject/application/dto"

type MockClientRepository struct {
    Clients map[string]*dto.ClientDTO
    Err     error
}

func NewMockClientRepository() *MockClientRepository {
    return &MockClientRepository{
        Clients: make(map[string]*dto.ClientDTO),
    }
}

func (m *MockClientRepository) GetByID(id string) (*dto.ClientDTO, error) {
    if m.Err != nil {
        return nil, m.Err
    }
    
    client, exists := m.Clients[id]
    if !exists {
        return nil, fmt.Errorf("client not found")
    }
    
    return client, nil
}

func (m *MockClientRepository) Save(client *dto.ClientDTO) error {
    if m.Err != nil {
        return m.Err
    }
    
    m.Clients[client.ID] = client
    return nil
}

func (m *MockClientRepository) Update(client *dto.ClientDTO) error {
    if m.Err != nil {
        return m.Err
    }
    
    if _, exists := m.Clients[client.ID]; !exists {
        return fmt.Errorf("client not found")
    }
    
    m.Clients[client.ID] = client
    return nil
}

// ... implement other methods
```

### Using Mocks in Use Case Tests
```go
func TestRentBookUseCase(t *testing.T) {
    // Setup mock repository
    mockClientRepo := NewMockClientRepository()
    mockClientRepo.Clients["client1"] = &dto.ClientDTO{
        ID:              "client1",
        Name:            "John Doe",
        MaxBooksAllowed: 3,
        BorrowedBookIDs: []string{},
    }
    
    // Inject mock into use case
    useCase := NewRentBookUseCase(mockClientRepo, mockBookRepo, ...)
    
    // Test
    err := useCase.Execute("client1", "book1", 14)
    assert.NoError(t, err)
}
```

## Why Define Interfaces in Application Layer?

### Dependency Inversion Principle
```
┌─────────────────────────────────────┐
│     Application Layer               │
│  ┌────────────┐  ┌───────────────┐ │
│  │ Use Cases  │→ │ Repositories  │ │ ← Interfaces defined
│  └────────────┘  │  (interfaces) │ │
│                  └───────────────┘ │
└──────────────────────┬──────────────┘
                       ↑ implements
┌──────────────────────┴──────────────┐
│     Infrastructure Layer            │
│  ┌─────────────────────────────┐   │
│  │ PostgreSQL Implementation   │   │
│  └─────────────────────────────┘   │
└─────────────────────────────────────┘
```

**Benefits:**
- Application doesn't depend on infrastructure
- Easy to swap implementations (PostgreSQL → MongoDB)
- Use cases can be tested with mocks
- Clear separation of concerns

## Common Repository Patterns

### Pagination
```go
type PageRequest struct {
    Page     int
    PageSize int
}

type PageResponse struct {
    Items      []*dto.ClientDTO
    Total      int
    Page       int
    TotalPages int
}

type ClientRepository interface {
    GetAllPaginated(req PageRequest) (*PageResponse, error)
}
```

### Filtering
```go
type ClientFilter struct {
    MinPoints   *int
    MaxPoints   *int
    HasOverdue  *bool
    Email       string
}

type ClientRepository interface {
    FindByFilter(filter ClientFilter) ([]*dto.ClientDTO, error)
}
```

### Sorting
```go
type SortOrder string

const (
    SortAsc  SortOrder = "ASC"
    SortDesc SortOrder = "DESC"
)

type ClientRepository interface {
    GetAllSorted(sortBy string, order SortOrder) ([]*dto.ClientDTO, error)
}
```

---

**Define your repository interfaces here - implement them in infrastructure!**

