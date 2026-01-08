# Application Use Cases

## Purpose

Use cases contain **application-specific business logic**. They orchestrate the flow of data between repositories and services to accomplish a specific task.

**Key Principle**: Use cases work with **application DTOs** (not domain entities or infrastructure types).

```go
// interfaces.go
package usecases

import "ExampleProject/application/dto"

// RentBookUseCase interface
type RentBookUseCase interface {
    Execute(clientID, bookID string, durationDays int) error
}

// ReturnBookUseCase interface
type ReturnBookUseCase interface {
    Execute(clientID, bookID string) error
}

// CheckPromotionsUseCase interface
type CheckPromotionsUseCase interface {
    Execute(clientID string) (*dto.PromotionResultDTO, error)
}

// GetClientBooksUseCase interface
type GetClientBooksUseCase interface {
    Execute(clientID string) ([]*dto.BookDTO, error)
}

// ConsumePointsUseCase interface
type ConsumePointsUseCase interface {
    Execute(clientID string, points int) error
}
```

## Rules

✅ **DO:**
- Implement one use case per file
- Inject dependencies via constructor
- Use **application DTOs** for data transfer
- Call repositories and services (via interfaces)
- Handle application flow and coordination
- Return application DTOs

❌ **DON'T:**
- Access database directly
- Make HTTP requests directly
- Include UI logic
- Depend on frameworks
- Mix multiple use cases in one file
- Use domain entities or infrastructure types directly

## Structure

```go
// Concrete implementation
type UseCaseNameImpl struct {
    // Dependencies (repositories and services)
    repository repositories.EntityRepository
    service    services.ExternalService
}

// Constructor returns interface type
func NewUseCaseName(repo repositories.EntityRepository, svc services.ExternalService) UseCaseName {
    return &UseCaseNameImpl{
        repository: repo,
        service:    svc,
    }
}

func (uc *UseCaseNameImpl) Execute(params...) (result, error) {
    // 1. Validate input
    // 2. Retrieve data
    // 3. Apply business rules
    // 4. Call external services
    // 5. Update entities
    // 6. Persist changes
    // 7. Return result
}
```

## Example: Rent Book Use Case

```go
// rent_book.go
package usecases

import (
    "ExampleProject/application/dto"
    "ExampleProject/application/repositories"
    "ExampleProject/application/services"
    "errors"
    "time"
)

// RentBookUseCaseImpl implements RentBookUseCase interface
type RentBookUseCaseImpl struct {
    clientRepo    repositories.ClientRepository
    bookRepo      repositories.BookRepository
    promotionSvc  services.PromotionService
    pointsService services.PointsService
}

// NewRentBookUseCase returns interface type for dependency injection
func NewRentBookUseCase(
    clientRepo repositories.ClientRepository,
    bookRepo repositories.BookRepository,
    promotionSvc services.PromotionService,
    pointsService services.PointsService,
) RentBookUseCase {  // Returns interface
    return &RentBookUseCaseImpl{
        clientRepo:    clientRepo,
        bookRepo:      bookRepo,
        promotionSvc:  promotionSvc,
        pointsService: pointsService,
    }
}

func (uc *RentBookUseCaseImpl) Execute(clientID, bookID string, durationDays int) error {
    // 1. Validate input
    if clientID == "" || bookID == "" {
        return errors.New("client ID and book ID are required")
    }
    if durationDays <= 0 || durationDays > 30 {
        return errors.New("duration must be between 1 and 30 days")
    }

    // 2. Retrieve client (returns application DTO)
    client, err := uc.clientRepo.GetByID(clientID)
    if err != nil {
        return errors.New("client not found")
    }

    // 3. Check business rule
    if len(client.BorrowedBookIDs) >= client.MaxBooksAllowed {
        return errors.New("client cannot borrow more books")
    }

    // 4. Retrieve book (returns application DTO)
    book, err := uc.bookRepo.GetByID(bookID)
    if err != nil {
        return errors.New("book not found")
    }

    // 5. Check business rule
    if !book.IsAvailable {
        return errors.New("book is not available")
    }

    // 6. Check for active promotions (external service, returns DTO)
    hasPromotion, err := uc.promotionSvc.HasActivePromotion(clientID)
    if err != nil {
        // Log error but don't fail the rent
        hasPromotion = false
    }

    // 7. Update DTOs with new state
    book.IsAvailable = false
    book.BorrowedBy = clientID
    dueDate := time.Now().AddDate(0, 0, durationDays)
    book.DueDate = &dueDate

    client.BorrowedBookIDs = append(client.BorrowedBookIDs, bookID)

    // 8. Add bonus points if has promotion
    if hasPromotion {
        bonusPoints := 10
        if err := uc.pointsService.AddPoints(clientID, bonusPoints); err != nil {
            // Log error but don't fail the rent
        }
    }

    // 9. Persist changes (repositories accept DTOs)
    if err := uc.bookRepo.Update(book); err != nil {
        return err
    }

    if err := uc.clientRepo.Update(client); err != nil {
        return err
    }

    return nil
}
```

## Example: Return Book Use Case

```go
// return_book.go
package usecases

import (
    "ExampleProject/domain/repositories"
    "ExampleProject/application/services"
    "errors"
    "fmt"
)

type ReturnBookUseCase struct {
    clientRepo      repositories.ClientRepository
    bookRepo        repositories.BookRepository
    pointsService   services.PointsService
    notificationSvc services.NotificationService
}

func NewReturnBookUseCase(
    clientRepo repositories.ClientRepository,
    bookRepo repositories.BookRepository,
    pointsService services.PointsService,
    notificationSvc services.NotificationService,
) *ReturnBookUseCase {
    return &ReturnBookUseCase{
        clientRepo:      clientRepo,
        bookRepo:        bookRepo,
        pointsService:   pointsService,
        notificationSvc: notificationSvc,
    }
}

func (uc *ReturnBookUseCase) Execute(clientID, bookID string) error {
    // 1. Get client
    client, err := uc.clientRepo.GetByID(clientID)
    if err != nil {
        return errors.New("client not found")
    }

    // 2. Get book
    book, err := uc.bookRepo.GetByID(bookID)
    if err != nil {
        return errors.New("book not found")
    }

    // 3. Validate book belongs to client
    if book.BorrowedBy != clientID {
        return errors.New("this book is not borrowed by the client")
    }

    // 4. Check if overdue
    isOverdue := book.IsOverdue()
    daysOverdue := book.DaysOverdue()

    // 5. Return book (business logic)
    if err := book.Return(); err != nil {
        return err
    }

    // 6. Remove from client's list
    client.RemoveBook(bookID)

    // 7. Handle points based on return status
    if isOverdue {
        // Deduct points for late return
        penalty := daysOverdue * 5
        if err := uc.pointsService.ConsumePoints(clientID, penalty); err != nil {
            // Log but continue
        }

        // Send notification
        message := fmt.Sprintf("Book '%s' was returned %d days late. %d points deducted.", 
            book.Title, daysOverdue, penalty)
        uc.notificationSvc.SendEmail(client.Email, "Late Return", message)
    } else {
        // Reward points for on-time return
        reward := 10
        if err := uc.pointsService.AddPoints(clientID, reward); err != nil {
            // Log but continue
        }

        // Send thank you notification
        message := fmt.Sprintf("Thank you for returning '%s' on time! You earned %d points.", 
            book.Title, reward)
        uc.notificationSvc.SendEmail(client.Email, "Book Returned", message)
    }

    // 8. Persist changes
    if err := uc.bookRepo.Update(book); err != nil {
        return err
    }

    if err := uc.clientRepo.Update(client); err != nil {
        return err
    }

    return nil
}
```

## Example: Check Client Promotions Use Case

```go
// check_promotions.go
package usecases

import (
    "ExampleProject/domain/repositories"
    "ExampleProject/application/services"
)

type CheckPromotionsUseCase struct {
    clientRepo   repositories.ClientRepository
    promotionSvc services.PromotionService
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

type PromotionResult struct {
    ClientName string
    ClientID   string
    Promotions []string
    HasActive  bool
}

func (uc *CheckPromotionsUseCase) Execute(clientID string) (*PromotionResult, error) {
    // 1. Get client from repository
    client, err := uc.clientRepo.GetByID(clientID)
    if err != nil {
        return nil, err
    }

    // 2. Check active promotions from external service
    hasActive, err := uc.promotionSvc.HasActivePromotion(clientID)
    if err != nil {
        return nil, err
    }

    // 3. Get all client promotions
    promotions, err := uc.promotionSvc.GetClientPromotions(clientID)
    if err != nil {
        return nil, err
    }

    // 4. Return aggregated result
    return &PromotionResult{
        ClientName: client.Name,
        ClientID:   client.ID,
        Promotions: promotions,
        HasActive:  hasActive,
    }, nil
}
```

## Example: Get Client Books Use Case

```go
// get_client_books.go
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
    // 1. Get client
    client, err := uc.clientRepo.GetByID(clientID)
    if err != nil {
        return nil, err
    }

    // 2. Get all books borrowed by client
    var books []*entities.Book
    for _, bookID := range client.BorrowedBooks {
        book, err := uc.bookRepo.GetByID(bookID)
        if err != nil {
            // Skip books that can't be found
            continue
        }
        books = append(books, book)
    }

    return books, nil
}
```

## Testing Use Cases

```go
// rent_book_test.go
package usecases_test

import (
    "testing"
    "ExampleProject/domain/entities"
    "ExampleProject/application/usecases"
)

// Mock repository
type MockClientRepository struct {
    client *entities.Client
    err    error
}

func (m *MockClientRepository) GetByID(id string) (*entities.Client, error) {
    return m.client, m.err
}

func (m *MockClientRepository) Update(client *entities.Client) error {
    m.client = client
    return m.err
}

// Test
func TestRentBook_Success(t *testing.T) {
    // Arrange
    client := &entities.Client{
        ID:              "1",
        Name:            "John",
        BorrowedBooks:   []string{},
        MaxBooksAllowed: 3,
        Points:          10,
    }
    
    book := &entities.Book{
        ID:          "book1",
        Title:       "Go Programming",
        IsAvailable: true,
    }
    
    mockClientRepo := &MockClientRepository{client: client}
    mockBookRepo := &MockBookRepository{book: book}
    mockPromotionSvc := &MockPromotionService{hasPromotion: true}
    mockPointsSvc := &MockPointsService{}
    
    useCase := usecases.NewRentBookUseCase(
        mockClientRepo,
        mockBookRepo,
        mockPromotionSvc,
        mockPointsSvc,
    )
    
    // Act
    err := useCase.Execute("1", "book1", 14)
    
    // Assert
    if err != nil {
        t.Errorf("Expected no error, got %v", err)
    }
    
    if len(client.BorrowedBooks) != 1 {
        t.Errorf("Expected 1 borrowed book, got %d", len(client.BorrowedBooks))
    }
    
    if book.IsAvailable {
        t.Error("Expected book to be unavailable")
    }
}
```

## Use Case Best Practices

1. **Single Responsibility**: One use case = one business operation
2. **Dependency Injection**: Pass all dependencies through constructor
3. **Return Domain Objects**: Return entities or simple types, not DTOs
4. **Error Handling**: Return meaningful errors
5. **Transaction Boundaries**: Use cases define transaction boundaries
6. **Idempotency**: Consider making operations idempotent when possible

---

**Place your use case files here - one file per use case!**

