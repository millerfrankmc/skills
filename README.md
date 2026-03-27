# KISS-DRY-YAGNI + Linus Torvalds

[![Skills](https://skills.sh/badge.svg)](https://skills.sh)

> **Code refactoring skill** applying simplicity principles: KISS (Keep It Simple, Stupid), DRY (Don't Repeat Yourself), YAGNI (You Ain't Gonna Need It) and the philosophy of Linus Torvalds.

## Installation

```bash
npx skills add millerfrankmc/skills/linus-kiss-dry-yagni
```

Supported on: Claude Code, Cursor, Codex, OpenCode, and [more agents](https://skills.sh).

---

## What this skill does

This skill transforms complex and over-engineered code into simple, direct, and maintainable code.

### Problems it solves

| Problem | Solution |
|----------|----------|
| Functions with 100+ lines | Split into functions ≤20 lines (≤50 for security) |
| Nesting of 6+ levels | Flat guard clauses |
| 10+ parameters in functions | Config object/struct |
| Interfaces with 1 implementation | Remove unnecessary abstraction (except security) |
| Duplicate code | Extract common function |
| Factories of factories | Simplify to direct functions |
| "Just in case" code | Remove (YAGNI) - except defense in depth |
| Security over-engineering | **Preserve** validations, headers, encryption |

### 🔒 Integrated Security

This skill **does not sacrifice security for simplicity**:

- ✅ Preserves input validations (multiple layers)
- ✅ Keeps SQL parameterized (never converts to interpolation)
- ✅ Preserves HTTP security headers
- ✅ Respects security functions (bcrypt, JWT, CSRF)
- ✅ Keeps critical comments (`// SECURITY:`, `// NEVER`)
- ✅ Does not apply DRY to validations with different threat models

> **"Simplicity must never sacrifice security."**

### Measured Results

In A/B tests against baseline without skill:

| Metric | Improvement |
|---------|--------|
| Pass rate | **+43%** (100% vs 57%) |
| Lines of code | **-42%** (simpler) |
| Refactoring time | **-9.6%** |

### 🔐 Security Benchmark

8 exhaustive evaluations verify that the skill **preserves critical protections**:

| Category | Evaluations | Pass Rate |
|-----------|--------------|-----------|
| Correct simplification | God Function, JWT Validation | 100% ✅ |
| Security preservation | Auth, SQL Injection, Input Validation | 100% ✅ |
| Contextual intelligence | DRY vs Context, Headers, Comments | 100% ✅ |
| **Total security verifications** | **40/40** | **100%** |

> See `/skill-creator-workspace/benchmark-results.md` for full details.

---

### 🚀 Performance Optimization

The skill includes language-specific performance guides:

| Language | Optimizations Covered |
|----------|-------------------------|
| **Python** | N+1 queries, list comprehensions, generators, asyncio |
| **TypeScript** | Memoization, lazy loading, bundle size, rendering |
| **Go** | Goroutines, buffers, allocations, profiling |
| **Kotlin** | Coroutines, lazy collections, flows, null-safety |
| **Rust** | Ownership, zero-copy, iterators, SIMD |

**Example - N+1 Queries (Python):**

```python
# BEFORE - N+1 problem
for user in users:
    orders = db.query(Order).filter_by(user_id=user.id).all()  # N queries

# AFTER - Single query
users_with_orders = db.query(User).options(joinedload(User.orders)).all()
```

See files in `/resources/performance/` for each language.

---

### 📚 Real Security Cases

The skill documents real vulnerabilities found in refactorings:

| Case | Vulnerability | Prevention |
|------|----------------|------------|
| **JWT Validation** | Removal of signature verification | Preserve complete validation |
| **SQL Injection** | Conversion to string interpolation | Maintain parameterization |
| **Password Hashing** | Reduction of bcrypt rounds | Preserve cost factor |
| **Input Validation** | Removal of "redundant" validations | Multiple layers of defense |
| **Security Headers** | Simplification of HTTP headers | Preserve all headers |

See `/resources/security/real-cases.md` for complete analysis.

---

## Usage

Once installed, the skill activates automatically when:

- You ask to "refactor code"
- You ask to "simplify code"
- You ask to "clean up code"
- You ask to "review code"
- Code with multiple obvious problems is detected

### Usage Example

**Input:**
```python
def process_user(user_id, user_name, user_email, user_phone, user_address,
                 user_city, user_country, is_active, is_premium, created_at):
    if user_id:
        if user_name:
            if user_email:
                if '@' in user_email and '.' in user_email:
                    if is_active:
                        # ... 50 more lines of nesting
```

**Output (automatic):**
```python
def process_user(user: User) -> Optional[dict]:
    if not user.id: return None
    if not user.name: return None
    if not is_valid_email(user.email): return None
    if not user.is_active: return None

    discount = 0.20 if user.is_premium else 0.10
    return build_user_data(user, discount)
```

---

## Applied Principles

### KISS - Keep It Simple, Stupid
- Small functions with single responsibility
- Code that reads from top to bottom
- No clever code (cryptic one-liners)

### DRY - Don't Repeat Yourself
- Extract duplicate code
- Centralize validations
- One change in one place

### YAGNI - You Ain't Gonna Need It
- Remove "just in case" code (except defense in depth security)
- No premature abstractions
- No interfaces with single implementation (except auth/crypto)

### Linus Torvalds Philosophy
> "The way to write clean code is to not write clever code."

- Guard clauses with early returns
- Specific error messages
- Obvious code over elegant code

---

## Concrete Limits (Non-Negotiable)

| Element | Limit | Action if exceeded |
|----------|--------|---------------------|
| **Lines per function** | 20 (50 for security) | Split into smaller functions |
| **Parameters per function** | 4 | Use object/struct |
| **Nesting levels** | 2 | Use guard clauses |
| **Code duplication** | 0 | Extract common function |
| **Unused interfaces** | 0 | Eliminate until needed |

---

## Included Resources

- **Common anti-patterns** - Identification and solutions
- **Security protection** - Guards that preserve validations and security controls
- **Language-specific security** - Python, TypeScript, Go, Kotlin, Rust
- **Language-specific performance** - Optimizations per language
- **Real security cases** - Documented vulnerabilities and prevention
- **Language examples** - Python, TypeScript, Go, Kotlin, Rust
- **Case studies** - Real refactorings
- **Simplification strategies** - Specific techniques
- **Linus Torvalds principles** - Philosophy behind simple code

---

## Detected Anti-Patterns

| Anti-Pattern | Warning Sign | Solution |
|-------------|-----------------|----------|
| **Pyramid of Doom** | 3+ levels of if/else | Guard clauses |
| **God Function** | Function does 3+ things | Split into functions |
| **Parameter Explosion** | 5+ parameters | Config object |
| **Interface Pollution** | Interface with 1 implementation | Use direct class |
| **Abstraction Addiction** | Factories of factories | Direct functions |
| **Future Proofing** | "Just in case" code | Remove |
| **Clever Code** | Cryptic one-liners | Expand to clear code |

---

## Case Studies

### Case 1: God Function (Python)
- **Before:** 80 lines, validates, calculates, saves, notifies
- **After:** 4 functions of 15 lines each
- **Benefit:** Testable, reusable, maintainable

### Case 2: Deep Nesting (Go)
- **Before:** 7 levels of if → impossible to follow
- **After:** Flat guard clauses → linear flow
- **Benefit:** Readable from top to bottom

### Case 3: Over-engineering (TypeScript)
- **Before:** Interface + Factory + Strategy for 2 options
- **After:** Simple if/else
- **Benefit:** 75% less code, clearer

---

## Repository Structure

```
.
├── LICENSE                     # MIT License
├── README.md                   # This file
└── linus-kiss-dry-yagni/       # Main skill
    ├── SKILL.md                # Skill definition
    ├── evals/
    │   └── evals.json          # A/B test cases
    └── resources/
        ├── anti-patterns/      # Common anti-patterns
        ├── cases/              # Case studies
        ├── decision/           # Decision frameworks
        ├── design/             # KISS-driven design
        ├── examples/           # Language examples
        ├── performance/        # Language optimizations (Python, TS, Go, Kotlin, Rust)
        ├── principles/         # Fundamental principles
        ├── security/           # Language security + real cases
        └── strategies/         # Simplification strategies
```

> **Note:** This `skills` repository contains multiple skills. Each skill is in its own folder.

---

## Testing

### A/B Testing - Simplification

- **5 test cases** (Python, TypeScript, Go, Kotlin, Ruby)
- **100% pass rate** with the skill vs 57% without
- **42% reduction** in average lines of code

### Security Benchmark

- **8 exhaustive evaluations** verify security preservation
- **40 security verifications** - 100% pass rate
- **0 unsafe simplifications** detected
- **Flexible names**: Accepts private (`_is_valid`) and public functions

### Performance Evaluations

- **Language-specific optimizations**
- **Prevention of N+1 queries**, memory leaks, unnecessary allocations
- **Preservation of critical performance code**

See `/skill-creator-workspace/benchmark-results.md` for detailed results.

---

## Contributing

Found a case where the skill doesn't work well? Open an issue with:

1. Input code
2. What you expected
3. What you got

---

## License

MIT

---

## Inspiration

> "Talk is cheap. Show me the code."
> — Linus Torvalds

This skill exists because simple code beats complex code. No more over-engineering. No more unnecessary abstractions. Just code that works and can be maintained.
