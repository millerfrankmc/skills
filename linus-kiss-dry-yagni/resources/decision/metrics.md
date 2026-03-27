# Metrics and Complexity Limits

These are the concrete limits that guide refactoring decisions.

## NON-Negotiable Limits

If these limits are exceeded, you MUST refactor:

| Element | Maximum Limit | Action When Exceeded |
|---------|---------------|----------------------|
| **Lines per function** | 20 | Split into smaller functions |
| **Parameters per function** | 4 | Use object/struct config |
| **Nesting levels** | 2 | Use guard clauses |
| **Classes per file** | 1 | Split into files |
| **Responsibilities per function** | 1 | Extract functions |
| **Responsibilities per class** | 1 | Split into classes |
| **Code duplication** | 0 | Extract common function |
| **Unused interfaces** | 0 | Remove until needed |
| **Comments explaining "what"** | 0 | Rename code |
| **Unused code** | 0 | Remove (it's in git) |

## Complexity Thresholds (Reference)

| Metric | Green ✅ | Yellow ⚠️ | Red ❌ |
|--------|----------|-----------|--------|
| **Lines per function** | 1-20 | 21-50 | 50+ |
| **Parameters per function** | 1-3 | 4 | 5+ |
| **Nesting levels** | 1-2 | 3 | 4+ |
| **Cyclomatic complexity** | 1-5 | 6-10 | 10+ |
| **Dependencies per file** | 1-5 | 6-10 | 10+ |
| **Functions per class** | 1-10 | 11-20 | 20+ |
| **Public methods per class** | 1-5 | 6-10 | 10+ |
| **Name length** | 5-30 chars | 31-50 | 50+ |

## How to Measure

### Lines of Code

```bash
# Python - count function lines
wc -l file.py

# JavaScript/TypeScript
npx cloc file.ts

# Go
gocloc file.go
```

### Cyclomatic Complexity

```bash
# Python
radon cc file.py -a

# JavaScript
npx complexity-report file.js

# Go
gocyclo file.go
```

### Nesting Levels

Manually count maximum levels of:
- `if` / `else` / `elif`
- `for` / `while`
- `try` / `catch` / `finally`
- `switch` / `case`
- Nested functions

### Parameters

```bash
# Count in Python
grep -E "^def \w+\(" file.py

# Count in TypeScript
grep -E "function \w+\(|\w+\(.*:" file.ts
```

## Decision Rules

### When to Extract a Function

Extract function if:
- [ ] Has >20 lines
- [ ] Does 2+ different things
- [ ] Has comment explaining a section
- [ ] Has code that could be reused
- [ ] Is difficult to name clearly

### When to Use a Config Object

Use object/struct if:
- [ ] Function has >4 parameters
- [ ] Parameters are optional
- [ ] Parameters belong to a concept
- [ ] They are frequently passed together

### When to Remove an Interface

Remove interface if:
- [ ] Has only 1 implementation
- [ ] No plans for second implementation
- [ ] Adds indirection without value

### When to Apply Guard Clauses

Apply guard clauses if:
- [ ] Has >2 nesting levels
- [ ] Function returns early in error cases
- [ ] Has validations at the beginning that nest the real code

## Calculation Examples

### Example 1: Function with 25 lines

```python
def process_user(data):  # Line 1
    if not data:         # Line 2
        return None      # Line 3

    validated = validate(data)   # Line 4
    if not validated:            # Line 5
        return None              # Line 6

    transformed = transform(validated)   # Line 7
    saved = save(transformed)            # Line 8
    notified = notify(saved)             # Line 9
    logged = log(notified)               # Line 10

    return logged        # Line 11
```

**Total**: 11 lines (✅ Within limit)

### Example 2: Function with 5 parameters

```python
# ❌ BEFORE: 5 parameters
def create_user(name, email, phone, address, is_active):
    pass

# ✅ AFTER: 1 object
def create_user(data: UserData):
    pass
```

### Example 3: 4 levels of nesting

```python
# ❌ BEFORE: 4 levels
def process(order):
    if order:                    # Level 1
        if order.valid:          # Level 2
            if order.payment:    # Level 3
                if order.paid:   # Level 4
                    fulfill(order)

# ✅ AFTER: 1 level
def process(order):
    if not order: return
    if not order.valid: return
    if not order.payment: return
    if not order.paid: return
    fulfill(order)
```

## Metrics Anti-Patterns

### Don't optimize metrics for metrics' sake

**Bad:**
```python
# Split function just to have fewer lines
def process_part1():
    # 10 lines
    pass

def process_part2():
    # 10 lines
    pass

def process_part3():
    # 10 lines
    pass
```

**Good:**
```python
# Split by responsibility
def validate():
    pass

def transform():
    pass

def save():
    pass
```

### Don't ignore context

Some functions legitimately need more lines:
- Configuration mappers
- Complex validators
- Mathematical algorithms

But they should be the exception, not the rule.

## Final Quality Metrics

After applying KISS-DRY-YAGNI, code should:

- [ ] Meet all complexity limits
- [ ] Be understandable without explanations
- [ ] Be testable without complex mocks
- [ ] Be modifiable without fear
- [ ] Have <5% code duplication
- [ ] Have 0 interfaces with 1 implementation
- [ ] Have 0 unused code
