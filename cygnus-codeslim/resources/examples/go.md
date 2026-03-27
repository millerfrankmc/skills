# Go Examples

## Anti-Clever Code

**BAD**:
```go
func process(s string) string {
    if r := regexp.MustCompile(`^(\w+)@(\w+)\.(\w+)$`); r.MatchString(s) {
        return strings.Map(func(r rune) rune { if r >= 'a' && r <= 'z' { return r - 32 }; return r }, s)
    }
    return ""
}
```

**GOOD**:
```go
func isValidEmail(s string) bool {
    return strings.Contains(s, "@") && strings.Contains(s, ".")
}

func toUpperCase(s string) string { return strings.ToUpper(s) }

func processEmail(email string) string {
    if !isValidEmail(email) { return "" }
    return toUpperCase(email)
}
```

---

## YAGNI - Remove Unnecessary Abstraction

**BAD**:
```go
type Processor interface { Process(data string) string }
type StringProcessor struct{}
func (p StringProcessor) Process(data string) string { return strings.ToUpper(data) }
```

**GOOD**:
```go
func processString(data string) string { return strings.ToUpper(data) }
```

---

## KISS - Guard Clauses

**GOOD** (guard clauses):
```go
func Process(r *Request) error {
    if r == nil { return errors.New("nil request") }
    if !r.Valid { return errors.New("invalid request") }
    if r.Data == "" { return errors.New("empty data") }
    return doSomething(r.Data)
}
```

---

## DRY - Centralize

**GOOD** (DRY with interface):
```go
type Customer interface { GetYearsActive() int }

func calculateDiscount(c Customer) float64 {
    if c.GetYearsActive() > 5 { return 0.2 }
    return 0.1
}
```
