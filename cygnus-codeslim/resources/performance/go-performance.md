# Performance Specific - Go

> Guide for performance optimizations in Go code. NEVER sacrifice obvious performance for "cleaner code".

---

## 1. Goroutines and Channels

### The Problem

```go
// ❌ INEFFICIENT - Goroutines without control
func processAll(items []Item) {
    for _, item := range items {
        go process(item) // Goroutine per item: thousands of goroutines!
    }
}

// ❌ INEFFICIENT - Unbuffered channel causes blocking
func producer() <-chan int {
    ch := make(chan int) // No buffer
    go func() {
        for i := 0; i < 1000; i++ {
            ch <- i // Blocks until someone receives
        }
        close(ch)
    }()
    return ch
}
```

### The Solution

```go
// ✅ CORRECT - Worker pool with controlled goroutines
func processAll(items []Item, workers int) {
    jobs := make(chan Item, len(items))
    results := make(chan Result, len(items))

    // Limited workers
    for w := 0; w < workers; w++ {
        go func() {
            for item := range jobs {
                results <- process(item)
            }
        }()
    }

    // Send jobs
    for _, item := range items {
        jobs <- item
    }
    close(jobs)

    // Collect results
    for i := 0; i < len(items); i++ {
        <-results
    }
}

// ✅ CORRECT - Channels with appropriate buffer
func producer() <-chan int {
    // Buffer allows producer to advance without blocking
    ch := make(chan int, 100)
    go func() {
        for i := 0; i < 1000; i++ {
            ch <- i // Doesn't block until buffer is full
        }
        close(ch)
    }()
    return ch
}

// ✅ CORRECT - Context for cancellation
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

## 2. sync.Pool for Object Reuse

```go
// ❌ INEFFICIENT - Creating objects repeatedly
func handleRequest(w http.ResponseWriter, r *http.Request) {
    buf := make([]byte, 4096) // New allocation per request
    // use buf...
}

// ✅ CORRECT - sync.Pool to reuse buffers
var bufferPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 4096)
    },
}

func handleRequest(w http.ResponseWriter, r *http.Request) {
    buf := bufferPool.Get().([]byte)
    defer bufferPool.Put(buf) // Return to pool when done

    // Reset buffer if needed
    buf = buf[:0]

    // use buf...
}

// ✅ CORRECT - Pool for complex structures
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

    // Reset state
    doc.Data = doc.Data[:0]
    for k := range doc.Meta {
        delete(doc.Meta, k)
    }

    // Process...
    doc.Data = append(doc.Data, input...)

    return doc
}
```

---

## 3. strings.Builder vs Concatenation

```go
// ❌ INEFFICIENT - Concatenation with +=
func buildString(parts []string) string {
    var result string
    for _, part := range parts {
        result += part // Creates new string each time
    }
    return result
}

// ✅ CORRECT - strings.Builder
import "strings"

func buildString(parts []string) string {
    var b strings.Builder
    // Pre-allocate if we know approximate size
    b.Grow(1024)

    for _, part := range parts {
        b.WriteString(part)
    }
    return b.String()
}

// ✅ CORRECT - Exact pre-allocation
func buildCSV(rows [][]string) string {
    // Calculate exact size
    totalSize := 0
    for _, row := range rows {
        for _, cell := range row {
            totalSize += len(cell) + 1 // +1 for separator
        }
    }

    var b strings.Builder
    b.Grow(totalSize) // Single allocation

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

## 4. Slice Pre-allocation

```go
// ❌ INEFFICIENT - Dynamic growth (multiple allocations)
func filterEven(nums []int) []int {
    var result []int
    for _, n := range nums {
        if n%2 == 0 {
            result = append(result, n) // Reallocation each time it grows
        }
    }
    return result
}

// ✅ CORRECT - Pre-allocate with estimated capacity
func filterEven(nums []int) []int {
    // Estimate that half will be even
    result := make([]int, 0, len(nums)/2)

    for _, n := range nums {
        if n%2 == 0 {
            result = append(result, n) // No reallocation
        }
    }
    return result
}

// ✅ CORRECT - make with length for specific cases
func initSlice(size int) []int {
    // If we know exact final size
    s := make([]int, size) // All elements exist (zero value)

    for i := range s {
        s[i] = i * i // Direct assignment, faster than append
    }
    return s
}

// ✅ CORRECT - copy for slices
func cloneSlice(s []int) []int {
    result := make([]int, len(s))
    copy(result, s) // Faster than manual loop
    return result
}
```

---

## 5. Escape Analysis

```go
// ❌ INEFFICIENT - Heap allocation (escapes)
func createUser(name string) *User {
    u := &User{Name: name} // Escapes to heap
    return u
}

// ✅ CORRECT - Stack allocation when possible
func createUser(name string) User {
    u := User{Name: name} // On stack
    return u // Copy, doesn't escape
}

// ❌ INEFFICIENT - Interface causes escape
func process(data interface{}) {
    // data escapes because interface{} is pointer
}

// ✅ CORRECT - Concrete types
func process(data []byte) {
    // data can stay on stack
}

// Check escapes: go build -gcflags="-m" .

// ❌ INEFFICIENT - Variable capture in closure
for i := 0; i < 10; i++ {
    go func() {
        fmt.Println(i) // i escapes to heap
    }()
}

// ✅ CORRECT - Pass as parameter
for i := 0; i < 10; i++ {
    go func(n int) {
        fmt.Println(n) // n on stack
    }(i)
}
```

---

## 6. Profiling with pprof

```go
// ✅ CORRECT - Enable profiling
import (
    "net/http"
    _ "net/http/pprof"
)

func main() {
    go func() {
        http.ListenAndServe("localhost:6060", nil)
    }()
    // Your application...
}

// Analysis commands:
// go tool pprof http://localhost:6060/debug/pprof/profile  # CPU
// go tool pprof http://localhost:6060/debug/pprof/heap     # Memory
// go tool pprof http://localhost:6060/debug/pprof/goroutine # Goroutines

// ✅ CORRECT - Benchmarks with testing
func BenchmarkProcessData(b *testing.B) {
    data := generateTestData(1000)

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        processData(data)
    }
}

// Run: go test -bench=. -benchmem

// ✅ CORRECT - Memory profile in benchmarks
func BenchmarkWithAllocs(b *testing.B) {
    b.ReportAllocs() // Report allocations

    for i := 0; i < b.N; i++ {
        result := processData()
        _ = result
    }
}
```

---

## 7. Struct vs Interface Efficiency

```go
// ❌ INEFFICIENT - Interface with overhead
func Sum(numbers []interface{}) int {
    total := 0
    for _, n := range numbers {
        total += n.(int) // Type assertion + boxing
    }
    return total
}

// ✅ CORRECT - Typed slice
func Sum(numbers []int) int {
    total := 0
    for _, n := range numbers {
        total += n // Direct, no overhead
    }
    return total
}

// ✅ CORRECT - Struct layout for cache

// ❌ Bad: unnecessary padding
type BadLayout struct {
    A bool     // 1 byte + 7 bytes padding
    B int64    // 8 bytes
    C bool     // 1 byte + 7 bytes padding
    D int64    // 8 bytes
} // Total: 32 bytes

// ✅ Good: fields grouped by size
type GoodLayout struct {
    B int64    // 8 bytes
    D int64    // 8 bytes
    A bool     // 1 byte
    C bool     // 1 byte
    // 6 bytes padding at end
} // Total: 24 bytes

// ✅ CORRECT - Reuse backing array when filtering
func filterInPlace(items []Item, predicate func(Item) bool) []Item {
    n := 0
    for _, item := range items {
        if predicate(item) {
            items[n] = item // Reuse same array
            n++
        }
    }
    return items[:n] // New slice, same backing array
}
```

---

## Go Specific Checklist

### Goroutines
- [ ] Worker pools to control concurrency
- [ ] Channels with appropriate buffer
- [ ] Context for timeout/cancellation
- [ ] Avoid race conditions with sync.Mutex

### Memory
- [ ] sync.Pool for frequently created objects
- [ ] Pre-allocate slices with make(capacity)
- [ ] strings.Builder for concatenation
- [ ] Minimize heap escapes

### Data
- [ ] Efficient struct layout (cache friendly)
- [ ] copy() instead of manual loops
- [ ] Reuse backing arrays when possible

### Profiling
- [ ] pprof to identify bottlenecks
- [ ] Benchmarks with -benchmem
- [ ] go build -gcflags="-m" to see escapes

### Concurrency
- [ ] sync.Map for concurrent maps
- [ ] sync/atomic for counters
- [ ] Consistent lock ordering to avoid deadlocks

---

## References

- [Go Performance](https://go.dev/wiki/Performance)
- [Go Data Structures](https://go.dev/blog/slices-intro)
- [Go Profiling](https://go.dev/blog/pprof)
- [High Performance Go](https://dave.cheney.net/high-performance-go)
