# Python Examples

## Anti-Clever Code

**BAD**:
```python
validate = lambda x: x and len(x) > 3 and x[0].isupper() or False
result = [x for x in data if validate(x)] or ["default"]
```

**GOOD**:
```python
def is_valid_name(name: str) -> bool:
    if not name:
        return False
    if len(name) <= 3:
        return False
    return name[0].isupper()

def filter_valid_names(names: list[str]) -> list[str]:
    valid = [name for name in names if is_valid_name(name)]
    return valid if valid else ["default"]
```

---

## YAGNI - Remove Unnecessary Abstraction

**BAD** (over-engineering):
```python
from abc import ABC, abstractmethod

class DataProcessor(ABC):
    @abstractmethod
    def process(self, data: dict) -> dict: pass

class UserDataProcessor(DataProcessor):
    def process(self, data: dict) -> dict:
        return {"name": data.get("name", "").upper()}
```

**GOOD** (YAGNI):
```python
def process_user(data: dict) -> dict:
    return {"name": data.get("name", "").upper()}
```

---

## KISS - Guard Clauses

**BAD** (nested):
```python
def process_user(user):
    if user:
        if user.is_active:
            if user.has_permission("write"):
                if user.quota > 0:
                    return perform_action(user)
    return None
```

**GOOD** (flat):
```python
def process_user(user):
    if not user: return None
    if not user.is_active: return None
    if not user.has_permission("write"): return None
    if user.quota <= 0: return None
    return perform_action(user)
```

---

## DRY - Centralize

**BAD** (duplicated):
```python
def validate_email(email: str) -> bool:
    return "@" in email and "." in email

def validate_user_email(user: dict) -> bool:
    email = user.get("email", "")
    return "@" in email and "." in email  # Duplicated
```

**GOOD** (DRY):
```python
def is_valid_email(email: str) -> bool:
    return "@" in email and "." in email

def validate_user_email(user: dict) -> bool:
    return is_valid_email(user.get("email", ""))
```

---

## Comments - WHAT vs WHY

**BAD** (WHAT comment):
```python
# Increment i by 1
i = i + 1

# Loop through users
for user in users:
    # Check if active
    if user.is_active:
        process(user)
```

**GOOD** (clean code or WHY):
```python
i += 1

for user in active_users:
    process(user)

# Timeout 30s because external service has p95 of 25s
timeout = 30
```
