# Infrastructure HTTP Clients

## Purpose

This folder contains **shared HTTP client utilities** and helpers for making external API calls. These utilities are used by service implementations.

## HTTP Client Interface

The HTTP client is defined as an interface to enable dependency injection and testing:

```go
// client_interface.go
package http

import (
    "context"
    "net/http"
)

// HTTPClient interface for dependency injection
// The standard library's *http.Client already implements this interface
type HTTPClient interface {
    Do(req *http.Request) (*http.Response, error)
}

// Verify that *http.Client implements HTTPClient
var _ HTTPClient = (*http.Client)(nil)
```

**Benefits:**
- ✅ Easy to mock in tests (no real HTTP calls)
- ✅ Can inject different implementations
- ✅ Standard library `*http.Client` implements it automatically
- ✅ Testable without network

**Usage in Services:**

```go
// infrastructure/loyalty_points/client.go
type LoyaltyClient struct {
    baseURL    string
    httpClient http.HTTPClient  // Interface instead of *http.Client
}

func NewLoyaltyClient(baseURL string, client http.HTTPClient) *LoyaltyClient {
    return &LoyaltyClient{
        baseURL:    baseURL,
        httpClient: client,
    }
}

// In production
realClient := &http.Client{Timeout: 10 * time.Second}
loyaltyClient := NewLoyaltyClient("http://api.example.com", realClient)

// In tests
mockClient := &MockHTTPClient{}
loyaltyClient := NewLoyaltyClient("http://api.example.com", mockClient)
```

## What Goes Here

- HTTP client configuration
- Common request/response handling
- Authentication helpers
- Retry logic
- Circuit breaker implementations
- Request logging/monitoring

## Example: Generic HTTP Client with Retry

```go
// external_client.go
package http

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "time"
)

type ExternalClient struct {
    client      *http.Client
    maxRetries  int
    retryDelay  time.Duration
    baseHeaders map[string]string
}

type ClientConfig struct {
    Timeout     time.Duration
    MaxRetries  int
    RetryDelay  time.Duration
    BaseHeaders map[string]string
}

func NewExternalClient(config ClientConfig) *ExternalClient {
    if config.Timeout == 0 {
        config.Timeout = 30 * time.Second
    }
    if config.MaxRetries == 0 {
        config.MaxRetries = 3
    }
    if config.RetryDelay == 0 {
        config.RetryDelay = time.Second
    }
    
    return &ExternalClient{
        client: &http.Client{
            Timeout: config.Timeout,
        },
        maxRetries:  config.MaxRetries,
        retryDelay:  config.RetryDelay,
        baseHeaders: config.BaseHeaders,
    }
}

// GET performs a GET request with retry logic
func (c *ExternalClient) GET(ctx context.Context, url string, headers map[string]string) (*http.Response, error) {
    return c.doWithRetry(ctx, "GET", url, nil, headers)
}

// POST performs a POST request with retry logic
func (c *ExternalClient) POST(ctx context.Context, url string, body interface{}, headers map[string]string) (*http.Response, error) {
    return c.doWithRetry(ctx, "POST", url, body, headers)
}

// PUT performs a PUT request with retry logic
func (c *ExternalClient) PUT(ctx context.Context, url string, body interface{}, headers map[string]string) (*http.Response, error) {
    return c.doWithRetry(ctx, "PUT", url, body, headers)
}

// DELETE performs a DELETE request with retry logic
func (c *ExternalClient) DELETE(ctx context.Context, url string, headers map[string]string) (*http.Response, error) {
    return c.doWithRetry(ctx, "DELETE", url, nil, headers)
}

// GetJSON performs a GET request and decodes JSON response
func (c *ExternalClient) GetJSON(ctx context.Context, url string, headers map[string]string, result interface{}) error {
    resp, err := c.GET(ctx, url, headers)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    
    if resp.StatusCode < 200 || resp.StatusCode >= 300 {
        body, _ := io.ReadAll(resp.Body)
        return fmt.Errorf("request failed with status %d: %s", resp.StatusCode, string(body))
    }
    
    return json.NewDecoder(resp.Body).Decode(result)
}

// PostJSON performs a POST request with JSON body and decodes response
func (c *ExternalClient) PostJSON(ctx context.Context, url string, body interface{}, headers map[string]string, result interface{}) error {
    resp, err := c.POST(ctx, url, body, headers)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    
    if resp.StatusCode < 200 || resp.StatusCode >= 300 {
        respBody, _ := io.ReadAll(resp.Body)
        return fmt.Errorf("request failed with status %d: %s", resp.StatusCode, string(respBody))
    }
    
    if result != nil {
        return json.NewDecoder(resp.Body).Decode(result)
    }
    
    return nil
}

// doWithRetry performs HTTP request with retry logic
func (c *ExternalClient) doWithRetry(ctx context.Context, method, url string, body interface{}, headers map[string]string) (*http.Response, error) {
    var lastErr error
    
    for attempt := 0; attempt < c.maxRetries; attempt++ {
        if attempt > 0 {
            select {
            case <-ctx.Done():
                return nil, ctx.Err()
            case <-time.After(c.retryDelay * time.Duration(attempt)):
            }
        }
        
        req, err := c.buildRequest(ctx, method, url, body, headers)
        if err != nil {
            return nil, err
        }
        
        resp, err := c.client.Do(req)
        if err != nil {
            lastErr = err
            continue
        }
        
        // Don't retry on client errors (4xx)
        if resp.StatusCode >= 400 && resp.StatusCode < 500 {
            return resp, nil
        }
        
        // Retry on server errors (5xx)
        if resp.StatusCode >= 500 {
            resp.Body.Close()
            lastErr = fmt.Errorf("server error: status %d", resp.StatusCode)
            continue
        }
        
        return resp, nil
    }
    
    return nil, fmt.Errorf("failed after %d attempts: %w", c.maxRetries, lastErr)
}

// buildRequest creates an HTTP request
func (c *ExternalClient) buildRequest(ctx context.Context, method, url string, body interface{}, headers map[string]string) (*http.Request, error) {
    var bodyReader io.Reader
    
    if body != nil {
        jsonData, err := json.Marshal(body)
        if err != nil {
            return nil, fmt.Errorf("failed to marshal body: %w", err)
        }
        bodyReader = bytes.NewBuffer(jsonData)
    }
    
    req, err := http.NewRequestWithContext(ctx, method, url, bodyReader)
    if err != nil {
        return nil, err
    }
    
    // Add base headers
    for key, value := range c.baseHeaders {
        req.Header.Set(key, value)
    }
    
    // Add request-specific headers
    for key, value := range headers {
        req.Header.Set(key, value)
    }
    
    if body != nil {
        req.Header.Set("Content-Type", "application/json")
    }
    req.Header.Set("Accept", "application/json")
    
    return req, nil
}
```

## Example: Authenticated Client

```go
// authenticated_client.go
package http

import (
    "context"
    "fmt"
)

type AuthenticatedClient struct {
    client *ExternalClient
    apiKey string
}

func NewAuthenticatedClient(baseURL, apiKey string) *AuthenticatedClient {
    client := NewExternalClient(ClientConfig{
        BaseHeaders: map[string]string{
            "X-API-Base-URL": baseURL,
        },
    })
    
    return &AuthenticatedClient{
        client: client,
        apiKey: apiKey,
    }
}

func (c *AuthenticatedClient) GET(ctx context.Context, url string) (*http.Response, error) {
    headers := c.authHeaders()
    return c.client.GET(ctx, url, headers)
}

func (c *AuthenticatedClient) POST(ctx context.Context, url string, body interface{}) (*http.Response, error) {
    headers := c.authHeaders()
    return c.client.POST(ctx, url, body, headers)
}

func (c *AuthenticatedClient) GetJSON(ctx context.Context, url string, result interface{}) error {
    headers := c.authHeaders()
    return c.client.GetJSON(ctx, url, headers, result)
}

func (c *AuthenticatedClient) PostJSON(ctx context.Context, url string, body interface{}, result interface{}) error {
    headers := c.authHeaders()
    return c.client.PostJSON(ctx, url, body, headers, result)
}

func (c *AuthenticatedClient) authHeaders() map[string]string {
    return map[string]string{
        "Authorization": fmt.Sprintf("Bearer %s", c.apiKey),
    }
}
```

## Example: Request Logger

```go
// logger.go
package http

import (
    "fmt"
    "net/http"
    "time"
)

type LoggingTransport struct {
    Transport http.RoundTripper
}

func (t *LoggingTransport) RoundTrip(req *http.Request) (*http.Response, error) {
    start := time.Now()
    
    fmt.Printf("[HTTP] %s %s\n", req.Method, req.URL.String())
    
    resp, err := t.transport().RoundTrip(req)
    
    duration := time.Since(start)
    
    if err != nil {
        fmt.Printf("[HTTP] %s %s - ERROR: %v (took %v)\n", 
            req.Method, req.URL.String(), err, duration)
        return nil, err
    }
    
    fmt.Printf("[HTTP] %s %s - %d (took %v)\n", 
        req.Method, req.URL.String(), resp.StatusCode, duration)
    
    return resp, nil
}

func (t *LoggingTransport) transport() http.RoundTripper {
    if t.Transport != nil {
        return t.Transport
    }
    return http.DefaultTransport
}

// Usage
func NewLoggingClient() *http.Client {
    return &http.Client{
        Transport: &LoggingTransport{},
    }
}
```

## Example: Rate Limiter

```go
// rate_limiter.go
package http

import (
    "context"
    "golang.org/x/time/rate"
    "net/http"
)

type RateLimitedTransport struct {
    Transport http.RoundTripper
    Limiter   *rate.Limiter
}

func (t *RateLimitedTransport) RoundTrip(req *http.Request) (*http.Response, error) {
    // Wait for rate limiter
    if err := t.Limiter.Wait(req.Context()); err != nil {
        return nil, err
    }
    
    return t.transport().RoundTrip(req)
}

func (t *RateLimitedTransport) transport() http.RoundTripper {
    if t.Transport != nil {
        return t.Transport
    }
    return http.DefaultTransport
}

// Usage: 10 requests per second
func NewRateLimitedClient(requestsPerSecond int) *http.Client {
    limiter := rate.NewLimiter(rate.Limit(requestsPerSecond), requestsPerSecond)
    
    return &http.Client{
        Transport: &RateLimitedTransport{
            Limiter: limiter,
        },
    }
}
```

## Best Practices

1. **Reuse HTTP Clients**: Create once, use many times
2. **Set Timeouts**: Always configure request timeouts
3. **Context**: Use context for cancellation and deadlines
4. **Retry Logic**: Implement exponential backoff
5. **Rate Limiting**: Respect API rate limits
6. **Logging**: Log all external calls
7. **Error Handling**: Distinguish between retryable and non-retryable errors
8. **Interface Usage**: Use HTTPClient interface for dependency injection

## Testing with Mock HTTP Client

```go
// mock_http_client.go (in tests)
package http_test

import (
    "bytes"
    "io"
    "net/http"
)

type MockHTTPClient struct {
    DoFunc func(req *http.Request) (*http.Response, error)
}

func (m *MockHTTPClient) Do(req *http.Request) (*http.Response, error) {
    if m.DoFunc != nil {
        return m.DoFunc(req)
    }
    // Default response
    return &http.Response{
        StatusCode: 200,
        Body:       io.NopCloser(bytes.NewBufferString(`{"success":true}`)),
    }, nil
}

// Usage in tests
func TestLoyaltyClient_GetPoints(t *testing.T) {
    mockClient := &MockHTTPClient{
        DoFunc: func(req *http.Request) (*http.Response, error) {
            // Verify request
            if req.URL.Path != "/api/v1/points/123" {
                t.Errorf("Wrong path: %s", req.URL.Path)
            }
            
            // Return mock response
            return &http.Response{
                StatusCode: 200,
                Body:       io.NopCloser(bytes.NewBufferString(`{"points":100}`)),
            }, nil
        },
    }
    
    client := NewLoyaltyClient("http://test.com", mockClient)
    points, err := client.GetPoints("123")
    
    if err != nil {
        t.Errorf("Unexpected error: %v", err)
    }
    if points != 100 {
        t.Errorf("Expected 100 points, got %d", points)
    }
}
```

---

**Place shared HTTP utilities here - used by all external service clients!**

