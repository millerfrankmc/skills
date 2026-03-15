# Seguridad Específica - Go

> Guía de protecciones de seguridad para código Go. NUNCA eliminar estos patrones al aplicar KISS-DRY-YAGNI.

---

## 1. Memory Safety

### Buffer Overflows
Go tiene memory safety por defecto, pero hay excepciones:

```go
// ❌ PROHIBIDO - Slice access sin bounds checking si se desactiva
// (En realidad Go siempre hace bounds checking, pero hay patrones riesgosos)

// ⚠️ CUIDADO - Uso de unsafe
import "unsafe"

func dangerous(data []byte) {
    // NUNCA hacer esto sin validación EXTREMA
    ptr := unsafe.Pointer(&data[0])
    // Manipulación de memoria raw
}

// ✅ CORRECTO - Siempre validar bounds
func safe(data []byte, index int) byte {
    if index < 0 || index >= len(data) {
        panic("index out of bounds") // O retornar error
    }
    return data[index]
}

// ✅ CORRECTO - Usar copy en lugar de manipulación manual
func copyData(dst, src []byte) int {
    return copy(dst, src) // Seguro, maneja bounds automáticamente
}
```

### Integer Overflow
```go
// ⚠️ CUIDADO - Integer overflow en operaciones aritméticas
func allocateBuffer(size int) []byte {
    // Si size es negativo o muy grande, podría causar problemas
    return make([]byte, size) // Panic si size < 0 o muy grande
}

// ✅ CORRECTO - Validar antes de asignar
func allocateBufferSafe(size int) ([]byte, error) {
    const maxSize = 100 * 1024 * 1024 // 100MB
    if size < 0 || size > maxSize {
        return nil, fmt.Errorf("invalid size: %d", size)
    }
    return make([]byte, size), nil
}
```

---

## 2. Goroutines y Concurrencia

### Race Conditions
```go
// ❌ PROHIBIDO - Data race
var counter int

func increment() {
    go func() {
        counter++ // Data race!
    }()
}

// ✅ CORRECTO - sync.Mutex
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

// ✅ CORRECTO - sync/atomic para operaciones simples
import "sync/atomic"

var counter int64

func incrementAtomic() {
    atomic.AddInt64(&counter, 1)
}

// ✅ CORRECTO - sync.Map para mapas concurrentes
var cache sync.Map

func getFromCache(key string) (interface{}, bool) {
    return cache.Load(key)
}
```

### Deadlocks
```go
// ❌ PROHIBIDO - Lock anidado sin orden
func transfer(from, to *Account, amount int) {
    from.mu.Lock()
    defer from.mu.Unlock()

    to.mu.Lock() // Deadlock posible si otra goroutine hace transfer(to, from)
    defer to.mu.Unlock()

    from.Balance -= amount
    to.Balance += amount
}

// ✅ CORRECTO - Orden de locks global
var (
    accountsMu sync.Mutex
    globalOrder = make(map[*Account]int) // Asignar orden a cada cuenta
)

func transferSafe(from, to *Account, amount int) error {
    // Siempre adquirir en orden de ID
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
// ❌ PROHIBIDO - Cerrar canal desde receiver
func bad() {
    ch := make(chan int)
    go func() {
        for v := range ch {
            if v == -1 {
                close(ch) // ❌ Solo el sender debe cerrar
            }
        }
    }()
}

// ✅ CORRECTO - Context para cancelación
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

    // Cancelar desde el coordinador
    cancel()
}
```

---

## 3. SQL Injection

```go
// ❌ PROHIBIDO - Concatenación de strings
func getUser(db *sql.DB, username string) (*User, error) {
    query := fmt.Sprintf("SELECT * FROM users WHERE username = '%s'", username)
    return db.Query(query)
}

// ✅ CORRECTO - Queries parametrizadas
func getUserSafe(db *sql.DB, username string) (*User, error) {
    row := db.QueryRow(
        "SELECT id, username, email FROM users WHERE username = ?",
        username,
    )
    // ...
}

// ✅ CORRECTO - sqlx para named parameters
import "github.com/jmoiron/sqlx"

func getUserNamed(db *sqlx.DB, username string) (*User, error) {
    var user User
    err := db.Get(&user,
        "SELECT * FROM users WHERE username = :username",
        map[string]interface{}{"username": username},
    )
    return &user, err
}

// ✅ CORRECTO - Squirrel para queries dinámicas
import sq "github.com/Masterminds/squirrel"

func buildQuery(userType string, active bool) (string, []interface{}, error) {
    return sq.Select("*").From("users").
        Where(sq.Eq{"type": userType}).
        Where(sq.Eq{"active": active}).
        ToSql() // Genera query parametrizada
}
```

---

## 4. Deserialización Segura

### JSON
```go
// ⚠️ CUIDADO - Deserialización en interface{} permite tipos inesperados
func processJSON(data []byte) error {
    var result interface{}
    json.Unmarshal(data, &result)
    // result podría ser cualquier cosa
    return nil
}

// ✅ CORRECTO - Estructura estricta con validación
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

    // Validación con go-playground/validator
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

// ❌ VULNERABLE A XXE por defecto
type Config struct {
    XMLName xml.Name `xml:"config"`
    Value   string   `xml:"value"`
}

// ✅ CORRECTO - Desactivar entidades externas
import (
    "encoding/xml"
    "io"
)

func parseXMLSafe(r io.Reader) (*Config, error) {
    decoder := xml.NewDecoder(r)
    decoder.Strict = true
    // En Go 1.10+, las entidades externas están desactivadas por defecto

    var config Config
    if err := decoder.Decode(&config); err != nil {
        return nil, err
    }
    return &config, nil
}
```

---

## 5. HTTP Seguro

### Servidor HTTP
```go
// ❌ PROHIBIDO - Servidor sin timeouts
srv := &http.Server{Addr: ":8080"}
srv.ListenAndServe() // Sin timeouts = resource exhaustion

// ✅ CORRECTO - Timeouts explícitos
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

// Esperar señal de terminación
quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
<-quit

// Shutdown graceful
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
if err := srv.Shutdown(ctx); err != nil {
    log.Fatal("Server forced to shutdown:", err)
}
```

### Cliente HTTP
```go
// ❌ PROHIBIDO - Cliente sin timeouts
resp, err := http.Get("https://api.example.com") // Puede colgarse para siempre

// ✅ CORRECTO - Cliente con timeouts y context
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

## 6. Cryptografía

### Password Hashing
```go
import "golang.org/x/crypto/bcrypt"

func hashPassword(password string) (string, error) {
    // Cost 12 es un buen balance (default es 10)
    bytes, err := bcrypt.GenerateFromPassword([]byte(password), 12)
    return string(bytes), err
}

func verifyPassword(password, hash string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
    return err == nil
}

// ✅ Alternativa - Argon2 (más moderno)
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

    // Almacenar salt + hash
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
        // Verificar algoritmo
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
// ❌ PROHIBIDO - Hardcodear secrets
const APIKey = "sk_live_1234567890"

// ✅ CORRECTO - Variables de entorno
import "os"

func getAPIKey() (string, error) {
    key := os.Getenv("API_KEY")
    if key == "" {
        return "", errors.New("API_KEY not set")
    }
    return key, nil
}

// ✅ CORRECTO - Cargar en startup y validar
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

// Inyectar config en handlers (no usar variable global)
type Server struct {
    config *Config
    db     *sql.DB
}
```

---

## 8. Validación de Input

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

    // Procesar...
}

// ✅ Validación adicional para casos especiales
func sanitizeFilename(name string) (string, error) {
    // Eliminar path traversal
    clean := filepath.Base(name)

    // Validar caracteres permitidos
    if matched, _ := regexp.MatchString(`^[a-zA-Z0-9._-]+$`, clean); !matched {
        return "", errors.New("invalid filename")
    }

    // Validar extensión
    ext := filepath.Ext(clean)
    allowed := map[string]bool{".txt": true, ".pdf": true, ".jpg": true}
    if !allowed[ext] {
        return "", errors.New("extension not allowed")
    }

    return clean, nil
}
```

---

## Checklist Go Específico

### Concurrencia
- [ ] Proteger datos compartidos con sync.Mutex o sync/atomic
- [ ] Usar sync.Map para caches concurrentes
- [ ] Evitar deadlocks con orden consistente de locks
- [ ] No cerrar channels desde receivers
- [ ] Usar context para cancelación de goroutines

### Seguridad
- [ ] SQL queries parametrizadas (?, no fmt.Sprintf)
- [ ] Validación estricta de JSON con structs
- [ ] HTTP server con timeouts (ReadTimeout, WriteTimeout)
- [ ] HTTP client con timeouts
- [ ] Graceful shutdown con signal handling
- [ ] bcrypt/argon2 para passwords
- [ ] Secrets en variables de entorno
- [ ] Validación de input con go-playground/validator
- [ ] No usar unsafe sin justificación extrema
- [ ] Validar bounds antes de make() con tamaños grandes

### Recursos
- [ ] Context with timeout para operaciones I/O
- [ ] Cerrar files/connections con defer
- [ ] Rate limiting en handlers HTTP

---

## Referencias

- [Go Security Cheat Sheet](https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Go_Security_Cheat_Sheet.md)
- [Go Secure Coding Practices](https://securego.io/)
- [Effective Go](https://go.dev/doc/effective_go)
