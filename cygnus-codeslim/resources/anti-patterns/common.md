# Common Anti-Patterns

Catalog of frequent problems and their immediate solutions.

## Index

1. [Pyramid of Doom](#pyramid-of-doom)
2. [God Function](#god-function)
3. [Parameter Explosion](#parameter-explosion)
4. [Interface Pollution](#interface-pollution)
5. [Abstraction Addiction](#abstraction-addiction)
6. [Future Proofing](#future-proofing)
7. [Clever Code](#clever-code)
8. [Comment Cancer](#comment-cancer)

---

## Pyramid of Doom

**Signs:** Multiple levels of nested if/else (>2 levels)

**Problem:** Impossible to follow code, hidden logic

**Before:**
```python
def process_order(order):
    if order:
        if order.valid:
            if order.payment:
                if order.payment.success:
                    if order.inventory:
                        return fulfill(order)
    return None
```

**After:**
```python
def process_order(order):
    if not order: return None
    if not order.valid: return None
    if not order.payment: return None
    if not order.payment.success: return None
    if not order.inventory: return None
    return fulfill(order)
```

---

## God Function

**Signs:** Function does 3+ different things, >20 lines

**Problem:** Difficult to test, reuse, and understand

**Before:**
```typescript
function processUser(data) {
  // Validate (10 lines)
  // Transform (15 lines)
  // Save to DB (10 lines)
  // Send email (10 lines)
  // Log (5 lines)
}
```

**After:**
```typescript
function processUser(data) {
  const validated = validateUser(data);
  const transformed = transformUser(validated);
  const saved = saveUser(transformed);
  notifyUser(saved);
  logActivity(saved);
  return saved;
}
```

---

## Parameter Explosion

**Signs:** Function with 5+ parameters

**Problem:** Difficult to call, easy to confuse order

**Before:**
```go
func CreateUser(name, email, phone, address, city, country, zip string, active bool, premium bool) error
```

**After:**
```go
type UserData struct {
    Name, Email, Phone string
    Address Address
    Active, Premium bool
}

func CreateUser(data UserData) error
```

---

## Interface Pollution

**Signs:** Interface with 1 implementation

**Problem:** Unnecessary complexity, harder to navigate code

**Before:**
```typescript
interface IUserRepository {
  findById(id: string): Promise<User>;
  save(user: User): Promise<User>;
}

class UserRepository implements IUserRepository {
  // single implementation
}
```

**After:**
```typescript
class UserRepository {
  async findById(id: string): Promise<User> { /* ... */ }
  async save(user: User): Promise<User> { /* ... */ }
}
```

---

## Abstraction Addiction

**Signs:** Factories of factories, services that only delegate

**Problem:** Indirection without value, impossible stack traces

**Before:**
```java
UserService -> UserRepositoryFactory -> UserRepositoryImpl
```

**After:**
```java
UserService -> UserRepository
```

---

## Future Proofing

**Signs:** Code "just in case we need it"

**Problem:** 80% never used, real complexity now

**Before:**
```python
class UserService:
    def create_user(self, data):
        # Support for multiple user types (only using one)
        # Support for pluggable validation (only using one)
        # Support for multiple backends (only using one)
        pass
```

**After:**
```python
def create_user(data):
    if not is_valid(data): raise ValueError("Invalid")
    return db.users.insert(data)
```

---

## Clever Code

**Signs:** One-liners, cryptic operators, "look how short"

**Problem:** Difficult to understand, debugging difficult

**Before:**
```python
result = data and [x for x in data if x.valid] or default
```

**After:**
```python
if not data:
    return default

valid_items = [x for x in data if x.valid]
return valid_items if valid_items else default
```

---

## Comment Cancer

**Signs:** Comments explaining "what" the code does

**Problem:** Comments lie, code doesn't

**Before:**
```python
# Increment the counter
i = i + 1

# Validate the email
if "@" in email and "." in email:
    # It's valid
    valid = True
```

**After:**
```python
counter += 1

if is_valid_email(email):
    valid = True
```

---

## Warning Signs Summary

| Anti-Pattern | Main Sign | Immediate Action |
|--------------|-----------|------------------|
| Pyramid of Doom | >2 levels of if | Guard clauses |
| God Function | >20 lines | Extract functions |
| Parameter Explosion | >4 parameters | Object config |
| Interface Pollution | 1 implementation | Remove interface |
| Abstraction Addiction | Factories of factories | Simplify |
| Future Proofing | Unused code | Remove |
| Clever Code | One-liners | Expand |
| Comment Cancer | // what it does | Rename |
