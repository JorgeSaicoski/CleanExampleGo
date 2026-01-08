# Data Transfer Objects (DTOs)

## Purpose

DTOs define the **structure of data** that flows in and out of your API. They separate your external API contract from your internal domain model.

## Rules

✅ **DO:**
- Define request and response structures
- Add JSON tags
- Include validation methods
- Add conversion functions (to/from domain entities)
- Document fields

❌ **DON'T:**
- Include business logic
- Use domain entities directly in API
- Mix DTO and domain entity concerns

## Example: Book DTOs

```go
// book_dto.go
package dto

import "errors"

// RentBookRequest represents the request to rent a book
type RentBookRequest struct {
    ClientID     string `json:"client_id"`
    BookID       string `json:"book_id"`
    DurationDays int    `json:"duration_days"`
}

func (r *RentBookRequest) Validate() error {
    if r.ClientID == "" {
        return errors.New("client_id is required")
    }
    if r.BookID == "" {
        return errors.New("book_id is required")
    }
    if r.DurationDays <= 0 || r.DurationDays > 30 {
        return errors.New("duration_days must be between 1 and 30")
    }
    return nil
}

// RentBookResponse represents the response from renting a book
type RentBookResponse struct {
    Success bool   `json:"success"`
    Message string `json:"message"`
}

// ReturnBookRequest represents the request to return a book
type ReturnBookRequest struct {
    ClientID string `json:"client_id"`
    BookID   string `json:"book_id"`
}

func (r *ReturnBookRequest) Validate() error {
    if r.ClientID == "" {
        return errors.New("client_id is required")
    }
    if r.BookID == "" {
        return errors.New("book_id is required")
    }
    return nil
}

// ReturnBookResponse represents the response from returning a book
type ReturnBookResponse struct {
    Success       bool   `json:"success"`
    Message       string `json:"message"`
    PointsEarned  int    `json:"points_earned,omitempty"`
    PointsDeducted int   `json:"points_deducted,omitempty"`
}

// BookResponse represents a book in API responses
type BookResponse struct {
    ID          string  `json:"id"`
    Title       string  `json:"title"`
    Author      string  `json:"author"`
    ISBN        string  `json:"isbn"`
    IsAvailable bool    `json:"is_available"`
    DueDate     *string `json:"due_date,omitempty"`
}

// BooksResponse represents a list of books
type BooksResponse struct {
    Books []BookResponse `json:"books"`
    Total int            `json:"total"`
}
```

## Example: Client DTOs

```go
// client_dto.go
package dto

import (
    "errors"
    "ExampleProject/domain/entities"
)

// ClientResponse represents a client in API responses
type ClientResponse struct {
    ID              string   `json:"id"`
    Name            string   `json:"name"`
    Email           string   `json:"email"`
    Points          int      `json:"points"`
    BorrowedBooks   []string `json:"borrowed_books"`
    MaxBooksAllowed int      `json:"max_books_allowed"`
    CanBorrow       bool     `json:"can_borrow"`
}

// ToClientResponse converts domain entity to DTO
func ToClientResponse(client *entities.Client) *ClientResponse {
    return &ClientResponse{
        ID:              client.ID,
        Name:            client.Name,
        Email:           client.Email,
        Points:          client.Points,
        BorrowedBooks:   client.BorrowedBooks,
        MaxBooksAllowed: client.MaxBooksAllowed,
        CanBorrow:       client.CanBorrow(),
    }
}

// PromotionsResponse represents promotions for a client
type PromotionsResponse struct {
    ClientID   string   `json:"client_id"`
    ClientName string   `json:"client_name"`
    Promotions []string `json:"promotions"`
    HasActive  bool     `json:"has_active"`
}

// ToPromotionsResponse converts use case result to DTO
func ToPromotionsResponse(result *usecases.PromotionResult) *PromotionsResponse {
    return &PromotionsResponse{
        ClientID:   result.ClientID,
        ClientName: result.ClientName,
        Promotions: result.Promotions,
        HasActive:  result.HasActive,
    }
}

// ConsumePointsRequest represents request to consume points
type ConsumePointsRequest struct {
    ClientID string `json:"client_id"`
    Points   int    `json:"points"`
}

func (r *ConsumePointsRequest) Validate() error {
    if r.ClientID == "" {
        return errors.New("client_id is required")
    }
    if r.Points <= 0 {
        return errors.New("points must be positive")
    }
    return nil
}

// ConsumePointsResponse represents response from consuming points
type ConsumePointsResponse struct {
    Success        bool `json:"success"`
    Message        string `json:"message"`
    RemainingPoints int `json:"remaining_points,omitempty"`
}
```

## Example: Generic DTOs

```go
// common_dto.go
package dto

// ErrorResponse represents an error response
type ErrorResponse struct {
    Error   bool   `json:"error"`
    Message string `json:"message"`
    Code    string `json:"code,omitempty"`
}

// SuccessResponse represents a generic success response
type SuccessResponse struct {
    Success bool   `json:"success"`
    Message string `json:"message"`
    Data    interface{} `json:"data,omitempty"`
}

// PaginationRequest represents pagination parameters
type PaginationRequest struct {
    Page     int `json:"page"`
    PageSize int `json:"page_size"`
}

func (p *PaginationRequest) Validate() error {
    if p.Page < 1 {
        p.Page = 1
    }
    if p.PageSize < 1 || p.PageSize > 100 {
        p.PageSize = 20
    }
    return nil
}

// PaginatedResponse represents a paginated response
type PaginatedResponse struct {
    Data       interface{} `json:"data"`
    Page       int         `json:"page"`
    PageSize   int         `json:"page_size"`
    TotalItems int         `json:"total_items"`
    TotalPages int         `json:"total_pages"`
}
```

## Conversion Functions

```go
// book_dto.go (continued)

import (
    "ExampleProject/domain/entities"
    "time"
)

// ToBookResponse converts domain entity to DTO
func ToBookResponse(book *entities.Book) *BookResponse {
    resp := &BookResponse{
        ID:          book.ID,
        Title:       book.Title,
        Author:      book.Author,
        ISBN:        book.ISBN,
        IsAvailable: book.IsAvailable,
    }
    
    if book.DueDate != nil {
        dueDateStr := book.DueDate.Format(time.RFC3339)
        resp.DueDate = &dueDateStr
    }
    
    return resp
}

// ToBooksResponse converts slice of entities to DTO
func ToBooksResponse(books []*entities.Book) *BooksResponse {
    bookResponses := make([]BookResponse, 0, len(books))
    
    for _, book := range books {
        bookResponses = append(bookResponses, *ToBookResponse(book))
    }
    
    return &BooksResponse{
        Books: bookResponses,
        Total: len(bookResponses),
    }
}

// ToBookEntity converts DTO to domain entity (for create/update)
func (r *CreateBookRequest) ToEntity() *entities.Book {
    return &entities.Book{
        Title:       r.Title,
        Author:      r.Author,
        ISBN:        r.ISBN,
        IsAvailable: true,
    }
}
```

## JSON Tags Explained

```go
type Example struct {
    // Basic tag
    Field1 string `json:"field1"`
    
    // Omit if empty
    Field2 string `json:"field2,omitempty"`
    
    // Ignore this field
    Field3 string `json:"-"`
    
    // Use field name as-is (even if uppercase)
    Field4 string `json:"Field4"`
    
    // Custom name with omitempty
    Field5 int `json:"custom_name,omitempty"`
}
```

## Validation Patterns

```go
// Simple validation
func (r *Request) Validate() error {
    if r.Field == "" {
        return errors.New("field is required")
    }
    return nil
}

// Multiple validations
func (r *Request) Validate() error {
    var errs []string
    
    if r.Email == "" {
        errs = append(errs, "email is required")
    }
    if !strings.Contains(r.Email, "@") {
        errs = append(errs, "email is invalid")
    }
    
    if len(errs) > 0 {
        return errors.New(strings.Join(errs, "; "))
    }
    
    return nil
}

// Using a validation library (recommended for complex validation)
import "github.com/go-playground/validator/v10"

type Request struct {
    Email string `json:"email" validate:"required,email"`
    Age   int    `json:"age" validate:"gte=0,lte=130"`
}

func (r *Request) Validate() error {
    validate := validator.New()
    return validate.Struct(r)
}
```

## API Documentation Example

```go
// CreateBookRequest represents the request to create a new book
//
// Example:
//   {
//     "title": "Clean Architecture",
//     "author": "Robert C. Martin",
//     "isbn": "978-0134494166"
//   }
type CreateBookRequest struct {
    // Title of the book (required, max 255 characters)
    Title string `json:"title" validate:"required,max=255"`
    
    // Author of the book (required, max 255 characters)
    Author string `json:"author" validate:"required,max=255"`
    
    // ISBN of the book (required, valid ISBN format)
    ISBN string `json:"isbn" validate:"required,isbn"`
}
```

## Best Practices

1. **Validation**: Always validate DTOs before passing to use cases
2. **Conversion**: Create explicit conversion functions
3. **Documentation**: Document fields and provide examples
4. **Versioning**: Consider versioning DTOs (v1, v2)
5. **Consistency**: Use consistent naming and structure
6. **Security**: Never expose sensitive data in responses
7. **Omitempty**: Use omitempty for optional fields

## Testing DTOs

```go
// client_dto_test.go
package dto_test

import (
    "testing"
    "ExampleProject/interfaces/dto"
)

func TestConsumePointsRequest_Validate(t *testing.T) {
    tests := []struct {
        name    string
        req     dto.ConsumePointsRequest
        wantErr bool
    }{
        {
            name: "valid request",
            req: dto.ConsumePointsRequest{
                ClientID: "123",
                Points:   10,
            },
            wantErr: false,
        },
        {
            name: "missing client ID",
            req: dto.ConsumePointsRequest{
                Points: 10,
            },
            wantErr: true,
        },
        {
            name: "negative points",
            req: dto.ConsumePointsRequest{
                ClientID: "123",
                Points:   -5,
            },
            wantErr: true,
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := tt.req.Validate()
            if (err != nil) != tt.wantErr {
                t.Errorf("Validate() error = %v, wantErr %v", err, tt.wantErr)
            }
        })
    }
}
```

---

**Define your DTOs here - keep your API contract separate from domain models!**

