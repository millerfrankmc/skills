# Simplification Strategies

Proven techniques to reduce complexity.

## 1. Guard Clauses

**Problem**: Deep nesting
**Solution**: Return early

```python
# Before
def process(user):
    if user:
        if user.active:
            if user.has_permission():
                return do_something(user)
    return None

# After
def process(user):
    if not user: return None
    if not user.active: return None
    if not user.has_permission(): return None
    return do_something(user)
```

## 2. Extract Function

**Problem**: Function does multiple things
**Solution**: Split into smaller functions

```python
# Before
def process_order(order):
    # validate
    # calculate totals
    # apply discounts
    # save
    # send email
    pass

# After
def process_order(order):
    validated = validate(order)
    totals = calculate_totals(validated)
    final_order = apply_discounts(totals)
    saved = save(final_order)
    send_confirmation(saved)
```

## 3. Replace Parameter with Object

**Problem**: Multiple parameters
**Solution**: Group into object/struct

```python
# Before
def create_user(name, email, phone, address, city, country, postal_code):
    pass

# After
@dataclass
class UserData:
    name: str
    email: str
    phone: str
    address: Address

def create_user(data: UserData):
    pass
```

## 4. Remove Premature Abstraction

**Problem**: Interface/abstract class with 1 implementation
**Solution**: Remove until needed

```python
# Before
class UserRepository(ABC):
    @abstractmethod
    def find(id): pass

class SqlUserRepository(UserRepository):
    def find(id): ...

# After
class UserRepository:
    def find(id): ...
```

## 5. Consolidate Conditional

**Problem**: Similar conditionals in multiple places
**Solution**: Extract to a function

```python
# Before
if user.age >= 18 and user.verified and not user.banned:
    # ...
# ... later
if user.age >= 18 and user.verified and not user.banned:
    # ...

# After
def can_access_premium(user):
    return user.age >= 18 and user.verified and not user.banned

if can_access_premium(user):
    # ...
```

## 6. Replace Nested Conditional with Guard Clauses

**Problem**: Deep if-else
**Solution**: Guard clauses with early returns

```typescript
// Before
function getPrice(user) {
    let price = 100;
    if (user.premium) {
        if (user.years > 5) {
            price = price * 0.7;
        } else {
            price = price * 0.8;
        }
    } else {
        if (user.years > 2) {
            price = price * 0.9;
        }
    }
    return price;
}

// After
function getPrice(user) {
    const BASE_PRICE = 100;

    if (!user.premium && user.years <= 2) return BASE_PRICE;
    if (!user.premium) return BASE_PRICE * 0.9;
    if (user.years > 5) return BASE_PRICE * 0.7;
    return BASE_PRICE * 0.8;
}
```

## 7. Remove Assignments to Parameters

**Problem**: Reassigning parameters
**Solution**: Use local variables

```python
# Before
def process(data):
    data = data.strip()
    data = data.lower()
    return data

# After
def process(raw_data):
    cleaned = raw_data.strip()
    normalized = cleaned.lower()
    return normalized
```

## Application Order

1. **First**: YAGNI → Remove unnecessary
2. **Second**: KISS → Simplify complex
3. **Third**: DRY → Centralize duplicates

This order avoids creating abstractions over things that should be removed.
