# Language-Specific Security - Go

> Security protection guide for Go code. NEVER remove these patterns when applying KISS-DRY-YAGNI.

---

## 1. Memory Safety

### Buffer Overflows
Go has memory safety by default, but there are exceptions:

```go
// ❌ FORBIDDEN - Slice access without bounds checking if disabled
// (Actually Go always does bounds checking, but there are risky patterns)

// ⚠️ CAUTION - Using unsafe
import "unsafe"

func dangerous(data []byte) {
    // NEVER do this without EXTREME validation
    ptr := unsafe.Pointer(&data[0])
    // Raw memory manipulation
}

// ✅ CORRECT - Always validate bounds
func safe(data []byte, index int) byte {
    if index < 0 || index >= len(data) {
        panic("index out of bounds") // Or return error
    }
    return data[index]
}

// ✅ CORRECT - Use copy instead of manual manipulation
func copyData(dst, src []byte) int {
    return copy(dst, src) // Safe, handles bounds automatically
}
```

### Integer Overflow
```go
// ⚠️ CAUTION - Integer overflow in arithmetic operations
func allocateBuffer(size int) []byte {
    // If size is negative or very large, it could cause problems
    return make([]byte, size) // Panic if size < 0 or too large
}

// ✅ CORRECT - Validate before allocating
func allocateBufferSafe(size int) ([]byte, error) {
    const maxSize = 100 * 1024 * 1024 // 100MB
    if size < 0 || size > maxSize {
        return nil, fmt.Errorf("invalid size: %d", size)
    }
    return make([]byte, size), nil
}
```

---

## 2. Goroutines and Concurrency

### Race Conditions
```go
// ❌ FORBIDDEN - Data race
var counter int

func increment() {
    go func() {
        counter++ // Data race!
    }()
}

// ✅ CORRECT - sync.Mutex
var (
    counter int
    mu      sync.Mutex
)

func incrementSafe() {
    go func() {
        mu.Lock()
        defer mu.Unlock()
        counter++
    }()
}

// ✅ CORRECT - sync/atomic for simple operations
import "sync/atomic"

var counter int64

func incrementAtomic() {
    atomic.AddInt64(&counter, 1)
}

// ✅ CORRECT - sync.Map for concurrent maps
var cache sync.Map

func getFromCache(key string) (interface{}, bool) {
    return cache.Load(key)
}
```

### Deadlocks
```go
// ❌ FORBIDDEN - Nested lock without order
func transfer(from, to *Account, amount int) {
    from.mu.Lock()
    defer from.mu.Unlock()

    to.mu.Lock() // Possible deadlock if another goroutine does transfer(to, from)
    defer to.mu.Unlock()

    from.Balance -= amount
    to.Balance += amount
}

// ✅ CORRECT - Global lock order
var (
    accountsMu sync.Mutex
    globalOrder = make(map[*Account]int) // Assign order to each account
)

func transferSafe(from, to *Account, amount int) error {
    // Always acquire in ID order
    first, second := from, to
    if globalOrder[from] > globalOrder[to] {
        first, second = to, from
    }

    first.mu.Lock()
    defer first.mu.Unlock()
    second.mu.Lock()
    defer second.mu.Unlock()

    if from.Balance < amount {
        return errors.New("insufficient funds")
    }

    from.Balance -= amount
    to.Balance += amount
    return nil
}
```

### Channel Safety
```go
// ❌ FORBIDDEN - Close channel from receiver
func bad() {
    ch := make(chan int)
    go func() {
        for v := range ch {
            if v == -1 {
                close(ch) // ❌ Only sender should close
            }
        }
    }()
}

// ✅ CORRECT - Context for cancellation
func good() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    ch := make(chan int)
    go func() {
        for {
            select {
            case v := <-ch:
                process(v)
            case <-ctx.Done():
                return
            }
        }
    }()

    // Cancel from coordinator
    cancel()
}
```

---

## 3. SQL Injection

```go
// ❌ FORBIDDEN - String concatenation
func getUser(db *sql.DB, username string) (*User, error) {
    query := fmt.Sprintf("SELECT * FROM users WHERE username = '%s'", username)
    return db.Query(query)
}

// ✅ CORRECT - Parameterized queries
func getUserSafe(db *sql.DB, username string) (*User, error) {
    row := db.QueryRow(
        "SELECT id, username, email FROM users WHERE username = ?",
        username,
    )
    // ...
}

// ✅ CORRECT - sqlx for named parameters
import "github.com/jmoiron/sqlx"

func getUserNamed(db *sqlx.DB, username string) (*User, error) {
    var user User
    err := db.Get(&user,
        "SELECT * FROM users WHERE username = :username",
        map[string]interface{}{"username": username},
    )
    return &user, err
}

// ✅ CORRECT - Squirrel for dynamic queries
import sq "github.com/Masterminds/squirrel"

func buildQuery(userType string, active bool) (string, []interface{}, error) {
    return sq.Select("*").From("users").
        Where(sq.Eq{"type": userType}).
        Where(sq.Eq{"active": active}).
        ToSql() // Generates parameterized query
}
```

---

## 4. Safe Deserialization

### JSON
```go
// ⚠️ CAUTION - Deserialization into interface{} allows unexpected types
func processJSON(data []byte) error {
    var result interface{}
    json.Unmarshal(data, &result)
    // result could be anything
    return nil
}

// ✅ CORRECT - Strict structure with validation
type UserInput struct {
    Name  string `json:"name" validate:"required,max=100"`
    Email string `json:"email" validate:"required,email"`
    Age   int    `json:"age" validate:"min=0,max=150"`
}

func processJSONSafe(data []byte) (*UserInput, error) {
    var input UserInput
    if err := json.Unmarshal(data, &input); err != nil {
        return nil, fmt.Errorf("invalid JSON: %w", err)
    }

    // Validation with go-playground/validator
    validate := validator.New()
    if err := validate.Struct(input); err != nil {
        return nil, fmt.Errorf("validation failed: %w", err)
    }

    return &input, nil
}
```

### XML (XXE Prevention)
```go
import "encoding/xml"

// ❌ VULNERABLE TO XXE by default
type Config struct {
    XMLName xml.Name `xml:"config"`
    Value   string   `xml:"value"`
}

// ✅ CORRECT - Disable external entities
import (
    "encoding/xml"
    "io"
)

func parseXMLSafe(r io.Reader) (*Config, error) {
    decoder := xml.NewDecoder(r)
    decoder.Strict = true
    // In Go 1.10+, external entities are disabled by default

    var config Config
    if err := decoder.Decode(&config); err != nil {
        return nil, err
    }
    return &config, nil
}
```

---

## 5. Secure HTTP

### HTTP Server
```go
// ❌ FORBIDDEN - Server without timeouts
srv := &http.Server{Addr: ":8080"}
srv.ListenAndServe() // No timeouts = resource exhaustion

// ✅ CORRECT - Explicit timeouts
srv := &http.Server{
    Addr:         ":8080",
    ReadTimeout:  5 * time.Second,
    WriteTimeout: 10 * time.Second,
    IdleTimeout:  120 * time.Second,
    MaxHeaderBytes: 1 << 20, // 1MB
    Handler:      handler,
}

// Graceful shutdown
go func() {
    if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
        log.Fatalf("listen: %s\n", err)
    }
}()

// Wait for termination signal
quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
<-quit

// Graceful shutdown
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
if err := srv.Shutdown(ctx); err != nil {
    log.Fatal("Server forced to shutdown:", err)
}
```

### HTTP Client
```go
// ❌ FORBIDDEN - Client without timeouts
resp, err := http.Get("https://api.example.com") // Can hang forever

// ✅ CORRECT - Client with timeouts and context
type Client struct {
    httpClient *http.Client
}

func NewClient() *Client {
    return &Client{
        httpClient: &http.Client{
            Timeout: 10 * time.Second,
            Transport: &http.Transport{
                TLSHandshakeTimeout:   5 * time.Second,
                ResponseHeaderTimeout: 5 * time.Second,
                MaxIdleConns:          100,
                MaxConnsPerHost:       100,
                IdleConnTimeout:       90 * time.Second,
            },
        },
    }
}

func (c *Client) Request(ctx context.Context, url string) (*http.Response, error) {
    ctx, cancel := context.WithTimeout(ctx, 10*time.Second)
    defer cancel()

    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, err
    }

    return c.httpClient.Do(req)
}
```

---

## 6. Cryptography

### Password Hashing
```go
import "golang.org/x/crypto/bcrypt"

func hashPassword(password string) (string, error) {
    // Cost 12 is a good balance (default is 10)
    bytes, err := bcrypt.GenerateFromPassword([]byte(password), 12)
    return string(bytes), err
}

func verifyPassword(password, hash string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
    return err == nil
}

// ✅ Alternative - Argon2 (more modern)
import "golang.org/x/crypto/argon2"

func hashPasswordArgon2(password string) string {
    salt := make([]byte, 16)
    rand.Read(salt)

    hash := argon2.IDKey(
        []byte(password),
        salt,
        1,      // iterations
        64*1024, // 64MB memory
        4,      // threads
        32,     // key length
    )

    // Store salt + hash
    return base64.StdEncoding.EncodeToString(salt) + "$" +
           base64.StdEncoding.EncodeToString(hash)
}
```

### JWT
```go
import "github.com/golang-jwt/jwt/v5"

type Claims struct {
    UserID string `json:"user_id"`
    jwt.RegisteredClaims
}

func createToken(userID string, secret []byte) (string, error) {
    claims := Claims{
        UserID: userID,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(24 * time.Hour)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            NotBefore: jwt.NewNumericDate(time.Now()),
        },
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(secret)
}

func verifyToken(tokenString string, secret []byte) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(token *jwt.Token) (interface{}, error) {
        // Verify algorithm
        if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
        }
        return secret, nil
    })

    if err != nil {
        return nil, err
    }

    if claims, ok := token.Claims.(*Claims); ok && token.Valid {
        return claims, nil
    }

    return nil, errors.New("invalid token")
}
```

---

## 7. Secrets Management

```go
// ❌ FORBIDDEN - Hardcoding secrets
const APIKey = "sk_live_1234567890"

// ✅ CORRECT - Environment variables
import "os"

func getAPIKey() (string, error) {
    key := os.Getenv("API_KEY")
    if key == "" {
        return "", errors.New("API_KEY not set")
    }
    return key, nil
}

// ✅ CORRECT - Load at startup and validate
type Config struct {
    APIKey    string
    DBURL     string
    JWTSecret []byte
}

func LoadConfig() (*Config, error) {
    cfg := &Config{
        APIKey:    os.Getenv("API_KEY"),
        DBURL:     os.Getenv("DATABASE_URL"),
        JWTSecret: []byte(os.Getenv("JWT_SECRET")),
    }

    if cfg.APIKey == "" {
        return nil, errors.New("API_KEY required")
    }
    if len(cfg.JWTSecret) < 32 {
        return nil, errors.New("JWT_SECRET must be at least 32 bytes")
    }

    return cfg, nil
}

// Inject config into handlers (don't use global variable)
type Server struct {
    config *Config
    db     *sql.DB
}
```

---

## 8. Input Validation

```go
import "github.com/go-playground/validator/v10"

type CreateUserRequest struct {
    Username string `json:"username" validate:"required,min=3,max=50,alphanum"`
    Email    string `json:"email" validate:"required,email,max=254"`
    Password string `json:"password" validate:"required,min=8,max=128"`
    Age      int    `json:"age" validate:"min=0,max=150"`
}

func (s *Server) CreateUser(w http.ResponseWriter, r *http.Request) {
    var req CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "invalid JSON", http.StatusBadRequest)
        return
    }

    validate := validator.New()
    if err := validate.Struct(req); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    // Process...
}

// ✅ Additional validation for special cases
func sanitizeFilename(name string) (string, error) {
    // Remove path traversal
    clean := filepath.Base(name)

    // Validate allowed characters
    if matched, _ := regexp.MatchString(`^[a-zA-Z0-9._-]+$`, clean); !matched {
        return "", errors.New("invalid filename")
    }

    // Validate extension
    ext := filepath.Ext(clean)
    allowed := map[string]bool{".txt": true, ".pdf": true, ".jpg": true}
    if !allowed[ext] {
        return "", errors.New("extension not allowed")
    }

    return clean, nil
}
```

---

## Go-Specific Checklist

### Concurrency
- [ ] Protect shared data with sync.Mutex or sync/atomic
- [ ] Use sync.Map for concurrent caches
- [ ] Avoid deadlocks with consistent lock ordering
- [ ] Don't close channels from receivers
- [ ] Use context for goroutine cancellation

### Security
- [ ] SQL parameterized queries (?, not fmt.Sprintf)
- [ ] Strict JSON validation with structs
- [ ] HTTP server with timeouts (ReadTimeout, WriteTimeout)
- [ ] HTTP client with timeouts
- [ ] Graceful shutdown with signal handling
- [ ] bcrypt/argon2 for passwords
- [ ] Secrets in environment variables
- [ ] Input validation with go-playground/validator
- [ ] Don't use unsafe without extreme justification
- [ ] Validate bounds before make() with large sizes

### Resources
- [ ] Context with timeout for I/O operations
- [ ] Close files/connections with defer
- [ ] Rate limiting in HTTP handlers

---

## References

- [Go Security Cheat Sheet](https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Go_Security_Cheat_Sheet.md)
- [Go Secure Coding Practices](https://securego.io/)
- [Effective Go](https://go.dev/doc/effective_go)
