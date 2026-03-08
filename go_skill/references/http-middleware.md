# HTTP & Middleware Engineering

## Table of Contents

- [Middleware Fundamentals](#middleware-fundamentals)
- [Interface-Based Middleware Chain](#interface-based-middleware-chain)
- [Common Middleware Patterns](#common-middleware-patterns)
- [HTTP Client Hygiene](#http-client-hygiene)
- [Request Context](#request-context)
- [Error Handling in HTTP](#error-handling-in-http)

## Middleware Fundamentals

### Basic Middleware Signature

```go
// Middleware wraps an http.Handler
type Middleware func(http.Handler) http.Handler

// Usage
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        log.Printf("%s %s %v", r.Method, r.URL.Path, time.Since(start))
    })
}
```

### Execution Order

```go
// Chain: A -> B -> C -> Handler -> C -> B -> A
//
// A (before) -> B (before) -> C (before) -> Handler
// A (after)  <- B (after)  <- C (after)  <- Handler

func Chain(h http.Handler, mws ...Middleware) http.Handler {
    // Apply in reverse order so first middleware runs first
    for i := len(mws) - 1; i >= 0; i-- {
        h = mws[i](h)
    }
    return h
}

// Usage
handler := Chain(
    finalHandler,
    loggingMiddleware,    // First: logs request
    authMiddleware,       // Second: checks auth
    rateLimitMiddleware,  // Third: rate limits
)
```

## Interface-Based Middleware Chain

### Reusable Middleware Chain

```go
type Chain struct {
    middlewares []Middleware
}

func NewChain(mws ...Middleware) *Chain {
    return &Chain{middlewares: mws}
}

func (c *Chain) Then(h http.Handler) http.Handler {
    for i := len(c.middlewares) - 1; i >= 0; i-- {
        h = c.middlewares[i](h)
    }
    return h
}

func (c *Chain) ThenFunc(fn http.HandlerFunc) http.Handler {
    return c.Then(fn)
}

func (c *Chain) Append(mws ...Middleware) *Chain {
    newMws := make([]Middleware, len(c.middlewares)+len(mws))
    copy(newMws, c.middlewares)
    copy(newMws[len(c.middlewares):], mws)
    return &Chain{middlewares: newMws}
}

// Usage
baseChain := NewChain(loggingMiddleware, recoveryMiddleware)
authChain := baseChain.Append(authMiddleware)

mux := http.NewServeMux()
mux.Handle("/public", baseChain.ThenFunc(publicHandler))
mux.Handle("/private", authChain.ThenFunc(privateHandler))
```

## Common Middleware Patterns

### Logging Middleware

```go
func LoggingMiddleware(logger *log.Logger) Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            start := time.Now()

            // Wrap response writer to capture status
            wrapped := &responseWriter{ResponseWriter: w, status: http.StatusOK}

            next.ServeHTTP(wrapped, r)

            logger.Printf(
                "%s %s %d %v",
                r.Method,
                r.URL.Path,
                wrapped.status,
                time.Since(start),
            )
        })
    }
}

type responseWriter struct {
    http.ResponseWriter
    status int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.status = code
    rw.ResponseWriter.WriteHeader(code)
}
```

### Recovery Middleware

```go
func RecoveryMiddleware(logger *log.Logger) Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            defer func() {
                if err := recover(); err != nil {
                    logger.Printf("panic: %v\n%s", err, debug.Stack())
                    http.Error(w, "Internal Server Error", http.StatusInternalServerError)
                }
            }()
            next.ServeHTTP(w, r)
        })
    }
}
```

### Timeout Middleware

```go
func TimeoutMiddleware(timeout time.Duration) Middleware {
    return func(next http.Handler) http.Handler {
        return http.TimeoutHandler(next, timeout, "Request timeout")
    }
}
```

### CORS Middleware

```go
func CORSMiddleware(allowedOrigins []string) Middleware {
    originSet := make(map[string]bool)
    for _, o := range allowedOrigins {
        originSet[o] = true
    }

    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            origin := r.Header.Get("Origin")
            if originSet[origin] || originSet["*"] {
                w.Header().Set("Access-Control-Allow-Origin", origin)
                w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
                w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
            }

            if r.Method == "OPTIONS" {
                w.WriteHeader(http.StatusNoContent)
                return
            }

            next.ServeHTTP(w, r)
        })
    }
}
```

### Rate Limiting Middleware

```go
import "golang.org/x/time/rate"

func RateLimitMiddleware(rps float64, burst int) Middleware {
    limiter := rate.NewLimiter(rate.Limit(rps), burst)

    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            if !limiter.Allow() {
                http.Error(w, "Too Many Requests", http.StatusTooManyRequests)
                return
            }
            next.ServeHTTP(w, r)
        })
    }
}

// Per-client rate limiting
func PerClientRateLimiter(rps float64, burst int) Middleware {
    clients := sync.Map{}

    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ip := r.RemoteAddr

            limiterI, _ := clients.LoadOrStore(ip, rate.NewLimiter(rate.Limit(rps), burst))
            limiter := limiterI.(*rate.Limiter)

            if !limiter.Allow() {
                http.Error(w, "Too Many Requests", http.StatusTooManyRequests)
                return
            }

            next.ServeHTTP(w, r)
        })
    }
}
```

## HTTP Client Hygiene

### Proper Client Configuration

```go
func NewHTTPClient() *http.Client {
    return &http.Client{
        Timeout: 30 * time.Second,
        Transport: &http.Transport{
            MaxIdleConns:        100,
            MaxIdleConnsPerHost: 10,
            MaxConnsPerHost:     100,
            IdleConnTimeout:     90 * time.Second,
            TLSHandshakeTimeout: 10 * time.Second,
            ExpectContinueTimeout: 1 * time.Second,
            ForceAttemptHTTP2:   true,
        },
    }
}

// ❌ Bad: Creates new client per request
func badFetch(url string) (*http.Response, error) {
    client := &http.Client{}  // No timeout!
    return client.Get(url)    // Connection not reused
}

// ✅ Good: Reuse client
var httpClient = NewHTTPClient()

func goodFetch(ctx context.Context, url string) (*http.Response, error) {
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, err
    }
    return httpClient.Do(req)
}
```

### Client Wrapper with Retry

```go
type HTTPClient struct {
    client     *http.Client
    maxRetries int
    baseDelay  time.Duration
}

func NewHTTPClientWithRetry(maxRetries int) *HTTPClient {
    return &HTTPClient{
        client:     NewHTTPClient(),
        maxRetries: maxRetries,
        baseDelay:  100 * time.Millisecond,
    }
}

func (c *HTTPClient) Do(req *http.Request) (*http.Response, error) {
    var lastErr error

    for attempt := 0; attempt <= c.maxRetries; attempt++ {
        if attempt > 0 {
            delay := c.baseDelay * time.Duration(1<<uint(attempt-1))
            select {
            case <-time.After(delay):
            case <-req.Context().Done():
                return nil, req.Context().Err()
            }
        }

        resp, err := c.client.Do(req)
        if err != nil {
            lastErr = err
            continue
        }

        // Retry on 5xx
        if resp.StatusCode >= 500 {
            resp.Body.Close()
            lastErr = fmt.Errorf("server error: %d", resp.StatusCode)
            continue
        }

        return resp, nil
    }

    return nil, fmt.Errorf("max retries exceeded: %w", lastErr)
}
```

### Response Body Handling

```go
// ✅ Always close response body
func fetchJSON(ctx context.Context, url string, result interface{}) error {
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return err
    }

    resp, err := httpClient.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()  // ALWAYS close

    if resp.StatusCode != http.StatusOK {
        // Read body for error message
        body, _ := io.ReadAll(io.LimitReader(resp.Body, 1024))
        return fmt.Errorf("unexpected status %d: %s", resp.StatusCode, body)
    }

    return json.NewDecoder(resp.Body).Decode(result)
}
```

## Request Context

### Context Value Pattern

```go
type contextKey string

const (
    userIDKey    contextKey = "userID"
    requestIDKey contextKey = "requestID"
)

func WithUserID(ctx context.Context, userID string) context.Context {
    return context.WithValue(ctx, userIDKey, userID)
}

func UserIDFromContext(ctx context.Context) (string, bool) {
    userID, ok := ctx.Value(userIDKey).(string)
    return userID, ok
}

// Middleware that sets context value
func RequestIDMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        requestID := r.Header.Get("X-Request-ID")
        if requestID == "" {
            requestID = uuid.NewString()
        }

        ctx := context.WithValue(r.Context(), requestIDKey, requestID)
        w.Header().Set("X-Request-ID", requestID)

        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

## Error Handling in HTTP

### Structured Error Responses

```go
type APIError struct {
    Code    string `json:"code"`
    Message string `json:"message"`
    Details any    `json:"details,omitempty"`
}

func writeError(w http.ResponseWriter, status int, err APIError) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(err)
}

// Handler with error handling
func userHandler(w http.ResponseWriter, r *http.Request) {
    user, err := getUser(r.Context(), r.URL.Query().Get("id"))
    if err != nil {
        if errors.Is(err, ErrNotFound) {
            writeError(w, http.StatusNotFound, APIError{
                Code:    "USER_NOT_FOUND",
                Message: "User not found",
            })
            return
        }
        writeError(w, http.StatusInternalServerError, APIError{
            Code:    "INTERNAL_ERROR",
            Message: "An unexpected error occurred",
        })
        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
}
```

### Error Handler Pattern

```go
type HandlerFunc func(w http.ResponseWriter, r *http.Request) error

func (fn HandlerFunc) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    if err := fn(w, r); err != nil {
        handleError(w, err)
    }
}

func handleError(w http.ResponseWriter, err error) {
    var apiErr *APIError
    if errors.As(err, &apiErr) {
        writeError(w, apiErr.Status, *apiErr)
        return
    }

    log.Printf("unhandled error: %v", err)
    writeError(w, http.StatusInternalServerError, APIError{
        Code:    "INTERNAL_ERROR",
        Message: "An unexpected error occurred",
    })
}

// Usage
mux.Handle("/users", HandlerFunc(func(w http.ResponseWriter, r *http.Request) error {
    user, err := getUser(r.Context(), r.URL.Query().Get("id"))
    if err != nil {
        return err  // Error handled by ServeHTTP
    }

    return json.NewEncoder(w).Encode(user)
}))
```
