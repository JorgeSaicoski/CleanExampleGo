# Loyalty Points System

## Purpose

This folder contains the implementation of the **Loyalty Points Service** - an external microservice that manages client loyalty points and discount eligibility.

**Architecture:** Infrastructure adapts the external Loyalty API to provide application DTOs.

## Structure

```
loyalty_points/
├── entities/                    ← Internal entities for Loyalty API responses
│   ├── points_response.go       ← Maps Loyalty API JSON response
│   └── transaction_response.go  ← Maps transaction API response
├── client.go                    ← HTTP client for loyalty API
└── service_impl.go              ← Implements application/services.PointsService
```

**Important:** 
- `entities/` are **internal** - they map external Loyalty API JSON format
- These convert to `application/dto` types
- Application never imports these entities

## How It Works

### 1. Application Defines What It Needs
```go
// application/services/points_service.go
type PointsService interface {
    GetBalance(clientID string) (*dto.PointsDTO, error)
}

// application/dto/points_dto.go
type PointsDTO struct {
    ClientID        string
    Points          int
    Tier            string
    HasDiscount     bool
    DiscountPercent int
}
```

### 2. Infrastructure Has Internal Entity for External Format
```go
// infrastructure/loyalty_points/entities/points_response.go (INTERNAL)
// This matches what the external Loyalty API returns
type PointsBalanceResponse struct {
    ClientID       string `json:"client_id"`
    TotalPoints    int    `json:"total_points"`
    CurrentBalance int    `json:"current_balance"`
    TierLevel      string `json:"tier_level"`
    HasActivePromo bool   `json:"has_active_promo"`
    PromoDiscount  int    `json:"promo_discount"`
    ExpirationDate string `json:"expiration_date"`
    // ... many other fields the external API returns
}

// Convert external API format to application DTO
func (r *PointsBalanceResponse) ToDTO() *dto.PointsDTO {
    return &dto.PointsDTO{
        ClientID:        r.ClientID,
        Points:          r.CurrentBalance,  // Map to application field
        Tier:            r.TierLevel,        // Map to application field
        HasDiscount:     r.HasActivePromo,
        DiscountPercent: r.PromoDiscount,
    }
}
```

### 3. Infrastructure Implements Interface
```go
// infrastructure/loyalty_points/service_impl.go
package loyalty_points

import (
    "ExampleProject/application/dto"
    "ExampleProject/application/services"
    "ExampleProject/infrastructure/loyalty_points/entities"
)

type LoyaltyPointsService struct {
    client  *HTTPClient
    baseURL string
}

// Implements application/services.PointsService
func NewLoyaltyPointsService(baseURL string) services.PointsService {
    return &LoyaltyPointsService{
        client:  NewHTTPClient(),
        baseURL: baseURL,
    }
}

// Returns application DTO
func (s *LoyaltyPointsService) GetBalance(clientID string) (*dto.PointsDTO, error) {
    // 1. Call external Loyalty API
    var response entities.PointsBalanceResponse
    err := s.client.Get(s.baseURL + "/api/loyalty/balance/" + clientID, &response)
    if err != nil {
        return nil, err
    }
    
    // 2. Convert external format to application DTO
    return response.ToDTO(), nil
}
```

## Use Cases

This service handles:
1. **Check if client has discount** - Based on loyalty tier
2. **Get client points balance** - Current available points
3. **Add points** - When client performs actions (rent books, etc.)
4. **Consume points** - When client redeems points
5. **Get transaction history** - View points history

## Use Cases

This service handles:
1. **Check if client has discount** - Based on loyalty tier
2. **Get client points balance** - Current available points
3. **Add points** - When client performs actions (rent books, etc.)
4. **Consume points** - When client redeems points
5. **Get transaction history** - View points history

## Internal Entities (External API Format)

These entities are **internal** to infrastructure and map the external Loyalty API JSON responses.

### PointsBalanceResponse
```go
// entities/points_response.go
package entities

import "ExampleProject/application/dto"

// Internal entity - maps external Loyalty API JSON response
type PointsBalanceResponse struct {
    ClientID          string  `json:"client_id"`
    TotalPoints       int     `json:"total_points"`
    CurrentBalance    int     `json:"current_balance"`
    TierLevel         string  `json:"tier_level"`
    HasActivePromo    bool    `json:"has_active_promo"`
    PromoDiscount     int     `json:"promo_discount"`
    ExpirationDate    string  `json:"expiration_date"`
    NextTierThreshold int     `json:"next_tier_threshold"`
    // ... other fields returned by external API
}

// Convert external API format to application DTO
func (r *PointsBalanceResponse) ToDTO() *dto.PointsDTO {
    return &dto.PointsDTO{
        ClientID:        r.ClientID,
        Points:          r.CurrentBalance,
        Tier:            r.TierLevel,
        HasDiscount:     r.HasActivePromo,
        DiscountPercent: r.PromoDiscount,
    }
}
```

### TransactionResponse
```go
// entities/transaction_response.go
package entities

import (
    "time"
    "ExampleProject/application/dto"
)

// Internal entity - maps external Loyalty API transaction response
type TransactionResponse struct {
    TransactionID string    `json:"transaction_id"`
    ClientID      string    `json:"client_id"`
    PointsAmount  int       `json:"points_amount"`
    TxType        string    `json:"type"` // "earn", "spend", "expired"
    Description   string    `json:"description"`
    Timestamp     time.Time `json:"timestamp"`
    BalanceBefore int       `json:"balance_before"`
    BalanceAfter  int       `json:"balance_after"`
}

// Convert to application DTO
func (t *TransactionResponse) ToDTO() *dto.PointsTransactionDTO {
    return &dto.PointsTransactionDTO{
        ID:        t.TransactionID,
        ClientID:  t.ClientID,
        Points:    t.PointsAmount,
        Type:      t.TxType,
        Reason:    t.Description,
        CreatedAt: t.Timestamp,
    }
}
```
package entities

import "time"

type PointsTransaction struct {
    ID        string
    ClientID  string
    Points    int
    Type      string // "earn" or "spend"
    Reason    string
    CreatedAt time.Time
}
```

## DTOs

### Check Discount
```go
// dto/check_discount_request.go
package dto

type CheckDiscountRequest struct {
    ClientID string `json:"client_id"`
}

// dto/check_discount_response.go
package dto

import "ExampleProject/infrastructure/loyalty_points/entities"

type CheckDiscountResponse struct {
    ClientID        string `json:"client_id"`
    HasDiscount     bool   `json:"has_discount"`
    DiscountPercent int    `json:"discount_percent"`
    Tier            string `json:"tier"`
    AvailablePoints int    `json:"available_points"`
}

func (r *CheckDiscountResponse) ToEntity() *entities.PointsBalance {
    return &entities.PointsBalance{
        ClientID:        r.ClientID,
        AvailablePoints: r.AvailablePoints,
        Tier:            r.Tier,
        HasDiscount:     r.HasDiscount,
        DiscountPercent: r.DiscountPercent,
    }
}
```

### Points Operations
```go
// dto/points_request.go
package dto

type AddPointsRequest struct {
    ClientID string `json:"client_id"`
    Points   int    `json:"points"`
    Reason   string `json:"reason"`
}

type ConsumePointsRequest struct {
    ClientID string `json:"client_id"`
    Points   int    `json:"points"`
}

// dto/points_response.go
package dto

type PointsResponse struct {
    Success  bool   `json:"success"`
    Message  string `json:"message"`
    Balance  int    `json:"balance"`
}
```

## Service Implementation

```go
// service_impl.go
package loyalty_points

import (
    "ExampleProject/application/services"
    "ExampleProject/infrastructure/loyalty_points/dto"
)

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

func (s *LoyaltyPointsService) AddPoints(clientID string, points int) error {
    req := &dto.AddPointsRequest{
        ClientID: clientID,
        Points:   points,
        Reason:   "book_rental",
    }
    return s.client.AddPoints(req)
}

func (s *LoyaltyPointsService) ConsumePoints(clientID string, points int) error {
    req := &dto.ConsumePointsRequest{
        ClientID: clientID,
        Points:   points,
    }
    return s.client.ConsumePoints(req)
}

// Additional method not in base interface
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
```

## HTTP Client

```go
// client.go
package loyalty_points

import (
    "bytes"
    "encoding/json"
    "fmt"
    "net/http"
    "time"
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
        client: &http.Client{
            Timeout: 10 * time.Second,
        },
    }
}

func (c *LoyaltyClient) GetBalance(clientID string) (*entities.PointsBalance, error) {
    url := fmt.Sprintf("%s/api/v1/points/balance/%s", c.baseURL, clientID)
    
    req, _ := http.NewRequest("GET", url, nil)
    c.addAuthHeaders(req)
    
    resp, err := c.client.Do(req)
    if err != nil {
        return nil, fmt.Errorf("failed to get balance: %w", err)
    }
    defer resp.Body.Close()
    
    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("unexpected status: %d", resp.StatusCode)
    }
    
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

func (c *LoyaltyClient) AddPoints(req *dto.AddPointsRequest) error {
    url := fmt.Sprintf("%s/api/v1/points/add", c.baseURL)
    return c.postJSON(url, req)
}

func (c *LoyaltyClient) ConsumePoints(req *dto.ConsumePointsRequest) error {
    url := fmt.Sprintf("%s/api/v1/points/consume", c.baseURL)
    return c.postJSON(url, req)
}

func (c *LoyaltyClient) postJSON(url string, payload interface{}) error {
    jsonData, _ := json.Marshal(payload)
    req, _ := http.NewRequest("POST", url, bytes.NewBuffer(jsonData))
    c.addAuthHeaders(req)
    req.Header.Set("Content-Type", "application/json")
    
    resp, err := c.client.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    
    if resp.StatusCode != http.StatusOK {
        return fmt.Errorf("request failed with status: %d", resp.StatusCode)
    }
    
    return nil
}

func (c *LoyaltyClient) addAuthHeaders(req *http.Request) {
    req.Header.Set("Authorization", "Bearer "+c.apiKey)
    req.Header.Set("Accept", "application/json")
}
```

## Usage in Use Cases

```go
// application/usecases/rent_book.go

func (uc *RentBookUseCase) Execute(clientID, bookID string) error {
    // ... get client and book ...
    
    // Check if client has discount
    hasDiscount, discountPercent, err := uc.loyaltyService.CheckDiscount(clientID)
    if err != nil {
        // Log but don't fail
        hasDiscount = false
    }
    
    if hasDiscount {
        // Apply discount to rental price
        // Add bonus points
        uc.loyaltyService.AddPoints(clientID, 20)
    } else {
        // Add regular points
        uc.loyaltyService.AddPoints(clientID, 10)
    }
    
    // ... complete rental ...
}
```

## Configuration

```go
// cmd/api/main.go

loyaltyService := loyalty_points.NewLoyaltyPointsService(
    os.Getenv("LOYALTY_SERVICE_URL"),    // "http://loyalty-api:8080"
    os.Getenv("LOYALTY_API_KEY"),        // API key for authentication
)
```

## API Endpoints (External Service)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/points/balance/:clientId` | Get client balance |
| POST | `/api/v1/points/add` | Add points |
| POST | `/api/v1/points/consume` | Consume points |
| POST | `/api/v1/discounts/check` | Check discount eligibility |
| GET | `/api/v1/points/history/:clientId` | Get transaction history |

## Benefits of This Structure

1. **Isolation**: All loyalty-related code in one place
2. **Type Safety**: Explicit entities and DTOs
3. **Testability**: Easy to mock the client
4. **Maintainability**: Changes to loyalty API only affect this folder
5. **Clear Boundaries**: Conversion between layers is explicit

---

**Implement the loyalty points integration following this pattern!**

