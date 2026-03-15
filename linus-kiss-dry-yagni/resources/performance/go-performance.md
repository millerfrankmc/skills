# Rendimiento Específico - Go

> Guía de optimizaciones de rendimiento para código Go. NUNCA sacrificar rendimiento obvio por "código más limpio".

---

## 1. Goroutines y Channels

### El Problema

```go
// ❌ INEFICIENTE - Goroutines sin control
func processAll(items []Item) {
    for _, item := range items {
        go process(item) // Goroutine por cada item: miles de goroutines!
    }
}

// ❌ INEFICIENTE - Channel sin buffer causa bloqueos
func producer() <-chan int {
    ch := make(chan int) // Sin buffer
    go func() {
        for i := 0; i < 1000; i++ {
            ch <- i // Bloquea hasta que alguien reciba
        }
        close(ch)
    }()
    return ch
}
```

### La Solución

```go
// ✅ CORRECTO - Worker pool con goroutines controladas
func processAll(items []Item, workers int) {
    jobs := make(chan Item, len(items))
    results := make(chan Result, len(items))

    // Workers limitados
    for w := 0; w < workers; w++ {
        go func() {
            for item := range jobs {
                results <- process(item)
            }
        }()
    }

    // Enviar trabajos
    for _, item := range items {
        jobs <- item
    }
    close(jobs)

    // Recoger resultados
    for i := 0; i < len(items); i++ {
        <-results
    }
}

// ✅ CORRECTO - Channels con buffer apropiado
func producer() <-chan int {
    // Buffer permite que el producer avance sin bloquearse
    ch := make(chan int, 100)
    go func() {
        for i := 0; i < 1000; i++ {
            ch <- i // No bloquea hasta que buffer esté lleno
        }
        close(ch)
    }()
    return ch
}

// ✅ CORRECTO - Context para cancelación
func processWithTimeout(ctx context.Context, items []Item) error {
    ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
    defer cancel()

    for _, item := range items {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
            if err := process(ctx, item); err != nil {
                return err
            }
        }
    }
    return nil
}
```

---

## 2. sync.Pool para Reuso de Objetos

```go
// ❌ INEFICIENTE - Crear objetos repetidamente
func handleRequest(w http.ResponseWriter, r *http.Request) {
    buf := make([]byte, 4096) // Nueva asignación por request
    // usar buf...
}

// ✅ CORRECTO - sync.Pool para reutilizar buffers
var bufferPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 4096)
    },
}

func handleRequest(w http.ResponseWriter, r *http.Request) {
    buf := bufferPool.Get().([]byte)
    defer bufferPool.Put(buf) // Devolver al pool al terminar

    // Resetear buffer si es necesario
    buf = buf[:0]

    // usar buf...
}

// ✅ CORRECTO - Pool para estructuras complejas
type Document struct {
    Data []byte
    Meta map[string]string
}

var docPool = sync.Pool{
    New: func() interface{} {
        return &Document{
            Data: make([]byte, 0, 1024),
            Meta: make(map[string]string),
        }
    },
}

func processDocument(input []byte) *Document {
    doc := docPool.Get().(*Document)

    // Resetear estado
    doc.Data = doc.Data[:0]
    for k := range doc.Meta {
        delete(doc.Meta, k)
    }

    // Procesar...
    doc.Data = append(doc.Data, input...)

    return doc
}
```

---

## 3. strings.Builder vs Concatenación

```go
// ❌ INEFICIENTE - Concatenación con +=
func buildString(parts []string) string {
    var result string
    for _, part := range parts {
        result += part // Crea nuevo string cada vez
    }
    return result
}

// ✅ CORRECTO - strings.Builder
import "strings"

func buildString(parts []string) string {
    var b strings.Builder
    // Pre-allocar si conocemos tamaño aproximado
    b.Grow(1024)

    for _, part := range parts {
        b.WriteString(part)
    }
    return b.String()
}

// ✅ CORRECTO - Pre-allocación exacta
func buildCSV(rows [][]string) string {
    // Calcular tamaño exacto
    totalSize := 0
    for _, row := range rows {
        for _, cell := range row {
            totalSize += len(cell) + 1 // +1 para separador
        }
    }

    var b strings.Builder
    b.Grow(totalSize) // Una sola asignación

    for i, row := range rows {
        if i > 0 {
            b.WriteByte('\n')
        }
        for j, cell := range row {
            if j > 0 {
                b.WriteByte(',')
            }
            b.WriteString(cell)
        }
    }
    return b.String()
}
```

---

## 4. Pre-allocación de Slices

```go
// ❌ INEFICIENTE - Crecimiento dinámico (múltiples allocations)
func filterEven(nums []int) []int {
    var result []int
    for _, n := range nums {
        if n%2 == 0 {
            result = append(result, n) // Reallocation cada vez que crece
        }
    }
    return result
}

// ✅ CORRECTO - Pre-allocar con capacidad estimada
func filterEven(nums []int) []int {
    // Estimamos que la mitad serán pares
    result := make([]int, 0, len(nums)/2)

    for _, n := range nums {
        if n%2 == 0 {
            result = append(result, n) // Sin reallocation
        }
    }
    return result
}

// ✅ CORRECTO - make con length para casos específicos
func initSlice(size int) []int {
    // Si sabemos el tamaño final exacto
    s := make([]int, size) // Todos los elementos existen (valor cero)

    for i := range s {
        s[i] = i * i // Asignación directa, más rápido que append
    }
    return s
}

// ✅ CORRECTO - copy para slices
func cloneSlice(s []int) []int {
    result := make([]int, len(s))
    copy(result, s) // Más rápido que loop manual
    return result
}
```

---

## 5. Escape Analysis

```go
// ❌ INEFICIENTE - Allocación en heap (escapa)
func createUser(name string) *User {
    u := &User{Name: name} // Escapa al heap
    return u
}

// ✅ CORRECTO - Stack allocation cuando es posible
func createUser(name string) User {
    u := User{Name: name} // En stack
    return u // Copia, no escapa
}

// ❌ INEFICIENTE - Interface causa escape
func process(data interface{}) {
    // data escapa porque interface{} es puntero
}

// ✅ CORRECTO - Tipos concretos
func process(data []byte) {
    // data puede quedarse en stack
}

// Verificar escapes: go build -gcflags="-m" .

// ❌ INEFICIENTE - Captura de variable en closure
for i := 0; i < 10; i++ {
    go func() {
        fmt.Println(i) // i escapa al heap
    }()
}

// ✅ CORRECTO - Pasar como parámetro
for i := 0; i < 10; i++ {
    go func(n int) {
        fmt.Println(n) // n en stack
    }(i)
}
```

---

## 6. Profiling con pprof

```go
// ✅ CORRECTO - Habilitar profiling
import (
    "net/http"
    _ "net/http/pprof"
)

func main() {
    go func() {
        http.ListenAndServe("localhost:6060", nil)
    }()
    // Tu aplicación...
}

// Comandos de análisis:
// go tool pprof http://localhost:6060/debug/pprof/profile  # CPU
// go tool pprof http://localhost:6060/debug/pprof/heap     # Memoria
// go tool pprof http://localhost:6060/debug/pprof/goroutine # Goroutines

// ✅ CORRECTO - Benchmarks con testing
func BenchmarkProcessData(b *testing.B) {
    data := generateTestData(1000)

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        processData(data)
    }
}

// Ejecutar: go test -bench=. -benchmem

// ✅ CORRECTO - Perfil de memoria en benchmarks
func BenchmarkWithAllocs(b *testing.B) {
    b.ReportAllocs() // Reporta allocations

    for i := 0; i < b.N; i++ {
        result := processData()
        _ = result
    }
}
```

---

## 7. Eficiencia de Structs vs Interfaces

```go
// ❌ INEFICIENTE - Interface con overhead
func Sum(numbers []interface{}) int {
    total := 0
    for _, n := range numbers {
        total += n.(int) // Type assertion + boxing
    }
    return total
}

// ✅ CORRECTO - Slice tipado
func Sum(numbers []int) int {
    total := 0
    for _, n := range numbers {
        total += n // Directo, sin overhead
    }
    return total
}

// ✅ CORRECTO - Struct layout para cache

// ❌ Mal: padding innecesario
type BadLayout struct {
    A bool     // 1 byte + 7 bytes padding
    B int64    // 8 bytes
    C bool     // 1 byte + 7 bytes padding
    D int64    // 8 bytes
} // Total: 32 bytes

// ✅ Bueno: campos agrupados por tamaño
type GoodLayout struct {
    B int64    // 8 bytes
    D int64    // 8 bytes
    A bool     // 1 byte
    C bool     // 1 byte
    // 6 bytes padding al final
} // Total: 24 bytes

// ✅ CORRECTO - Reusar backing array al filtrar
func filterInPlace(items []Item, predicate func(Item) bool) []Item {
    n := 0
    for _, item := range items {
        if predicate(item) {
            items[n] = item // Reusar mismo array
            n++
        }
    }
    return items[:n] // Nuevo slice, mismo backing array
}
```

---

## Checklist Go Específico

### Goroutines
- [ ] Worker pools para controlar concurrencia
- [ ] Channels con buffer apropiado
- [ ] Context para timeout/cancelación
- [ ] Evitar race conditions con sync.Mutex

### Memoria
- [ ] sync.Pool para objetos frecuentemente creados
- [ ] Pre-allocar slices con make(capacity)
- [ ] strings.Builder para concatenación
- [ ] Minimizar escapes al heap

### Datos
- [ ] Struct layout eficiente (cache friendly)
- [ ] copy() en lugar de loops manuales
- [ ] Reusar backing arrays cuando sea posible

### Profiling
- [ ] pprof para identificar cuellos de botella
- [ ] Benchmarks con -benchmem
- [ ] go build -gcflags="-m" para ver escapes

### Concurrency
- [ ] sync.Map para mapas concurrentes
- [ ] sync/atomic para contadores
- [ ] Orden consistente de locks para evitar deadlocks

---

## Referencias

- [Go Performance](https://go.dev/wiki/Performance)
- [Go Data Structures](https://go.dev/blog/slices-intro)
- [Go Profiling](https://go.dev/blog/pprof)
- [High Performance Go](https://dave.cheney.net/high-performance-go)
