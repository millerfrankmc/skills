# Ejemplos Python

## Anti-Clever Code

**MAL**:
```python
validate = lambda x: x and len(x) > 3 and x[0].isupper() or False
result = [x for x in data if validate(x)] or ["default"]
```

**BIEN**:
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

## YAGNI - Eliminar Abstracción Innecesaria

**MAL** (sobre-ingeniería):
```python
from abc import ABC, abstractmethod

class DataProcessor(ABC):
    @abstractmethod
    def process(self, data: dict) -> dict: pass

class UserDataProcessor(DataProcessor):
    def process(self, data: dict) -> dict:
        return {"name": data.get("name", "").upper()}
```

**BIEN** (YAGNI):
```python
def process_user(data: dict) -> dict:
    return {"name": data.get("name", "").upper()}
```

---

## KISS - Guard Clauses

**MAL** (anidado):
```python
def process_user(user):
    if user:
        if user.is_active:
            if user.has_permission("write"):
                if user.quota > 0:
                    return perform_action(user)
    return None
```

**BIEN** (plano):
```python
def process_user(user):
    if not user: return None
    if not user.is_active: return None
    if not user.has_permission("write"): return None
    if user.quota <= 0: return None
    return perform_action(user)
```

---

## DRY - Centralizar

**MAL** (duplicado):
```python
def validate_email(email: str) -> bool:
    return "@" in email and "." in email

def validate_user_email(user: dict) -> bool:
    email = user.get("email", "")
    return "@" in email and "." in email  # Duplicado
```

**BIEN** (DRY):
```python
def is_valid_email(email: str) -> bool:
    return "@" in email and "." in email

def validate_user_email(user: dict) -> bool:
    return is_valid_email(user.get("email", ""))
```

---

## Comentarios - QUÉ vs POR QUÉ

**MAL** (comentario QUÉ):
```python
# Increment i by 1
i = i + 1

# Loop through users
for user in users:
    # Check if active
    if user.is_active:
        process(user)
```

**BIEN** (código limpio o POR QUÉ):
```python
i += 1

for user in active_users:
    process(user)

# Timeout 30s porque el servicio externo tiene p95 de 25s
timeout = 30
```
