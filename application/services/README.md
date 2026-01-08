# Application Services (Interfaces)

**Define your service interfaces here – implement them in `infrastructure/`!**

---

## Purpose

Service interfaces represent contracts for external systems (third-party APIs, microservices, domain services) or complex business operations. They are defined in the **application layer** and implemented in the **infrastructure layer**.

**Key Principle**: The application defines **what it expects**, and the infrastructure adapts to provide it.

---

## Rules

✅ **DO:**
- Keep interfaces focused and cohesive
- Use **application DTOs** in method signatures
- Document expected behavior
- Define interfaces for external services
- Return domain errors (not HTTP errors)
- Include context.Context for production code

❌ **DON'T:**
- Depend on external libraries in the interface
- Mix multiple service concerns
- Use HTTP-specific types in interface
- Use infrastructure-specific types (like database entities)
- Implement the service here (implement in infrastructure)

---

## Example: Email Service (Resend)

Suppose you want to send transactional emails using [Resend](https://resend.com/):

### Interface Definition

```go
// email_service.go
package services

import "context"

// EmailService handles sending emails via Resend or other providers
// Only domain types and context.Context in the interface
// No external library types

type EmailService interface {
    SendEmail(ctx context.Context, to, subject, body string) error
    SendBulkEmail(ctx context.Context, recipients []string, subject, body string) error
}
```

### Implementation Hint (infrastructure/services/email_service_impl.go)

```go
// EmailServiceImpl implements EmailService using Resend
// This is just a sketch for documentation purposes
package services

import (
    "context"
    "github.com/resend/resend-go" // External library only in implementation
)

type EmailServiceImpl struct {
    client *resend.Client
}

func NewEmailService(apiKey string) EmailService {
    client := resend.NewClient(apiKey)
    return &EmailServiceImpl{client: client}
}

func (s *EmailServiceImpl) SendEmail(ctx context.Context, to, subject, body string) error {
    // Use Resend client to send email
    // ...
    return nil
}

func (s *EmailServiceImpl) SendBulkEmail(ctx context.Context, recipients []string, subject, body string) error {
    // Use Resend client to send bulk email
    // ...
    return nil
}
```

---

## Other Service Interface Examples

### Points Service
```go
// points_service.go
package services

import (
    "context"
    "ExampleProject/application/dto"
)

type PointsService interface {
    GetBalance(ctx context.Context, clientID string) (*dto.PointsDTO, error)
    AddPoints(ctx context.Context, clientID string, points int) error
    ConsumePoints(ctx context.Context, clientID string, points int) error
    GetHistory(ctx context.Context, clientID string) ([]*dto.PointsTransactionDTO, error)
}
```

### Promotion Service
```go
// promotion_service.go
package services

import (
    "context"
    "ExampleProject/application/dto"
)

type PromotionService interface {
    HasActivePromotion(ctx context.Context, clientID string) (bool, error)
    GetClientPromotions(ctx context.Context, clientID string) ([]*dto.PromotionDTO, error)
    ApplyPromotion(ctx context.Context, clientID, promotionID string) error
    ValidatePromotion(ctx context.Context, promotionCode string) (bool, error)
}
```

### Search Service
```go
// search_service.go
package services

import (
    "context"
    "ExampleProject/application/dto"
)

type SearchService interface {
    SearchBooks(ctx context.Context, query string, limit int) ([]*dto.BookDTO, error)
    SearchClients(ctx context.Context, query string, limit int) ([]*dto.ClientDTO, error)
    IndexBook(ctx context.Context, book *dto.BookDTO) error
    RemoveBookFromIndex(ctx context.Context, bookID string) error
}
```

---

## Testing with Service Interfaces

Create mocks for testing:

```go
// mock_promotion_service.go
package services_test

type MockPromotionService struct {
    HasPromotionResult bool
    PromotionsList    []string
    Err               error
}

func (m *MockPromotionService) HasActivePromotion(ctx context.Context, clientID string) (bool, error) {
    return m.HasPromotionResult, m.Err
}
func (m *MockPromotionService) GetClientPromotions(ctx context.Context, clientID string) ([]string, error) {
    return m.PromotionsList, m.Err
}
func (m *MockPromotionService) ApplyPromotion(ctx context.Context, clientID, promotionID string) error {
    return m.Err
}
func (m *MockPromotionService) ValidatePromotion(ctx context.Context, promotionCode string) (bool, error) {
    return true, m.Err
}
```

---

## Service vs Repository

- **Repository:** Data access abstraction, CRUD, always returns domain entities
- **Service:** Business operation abstraction, may coordinate multiple entities, may call external APIs, may return domain entities or other types

---

## Error Handling

Services should return domain errors (not HTTP errors):

```go
var (
    ErrServiceUnavailable = errors.New("external service unavailable")
    ErrTimeout            = errors.New("service request timeout")
    ErrInvalidResponse    = errors.New("invalid response from service")
)
```

---

## Summary

- Define service interfaces in the application layer
- Implement them in the infrastructure layer
- Use domain types and context.Context
- Return domain errors
- Mock interfaces for testing
- Keep interfaces focused and cohesive

---

**Place your service interfaces here!**
