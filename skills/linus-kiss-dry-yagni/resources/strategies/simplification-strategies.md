# Estrategias de Simplificación

Técnicas probadas para reducir complejidad.

## 1. Guard Clauses

**Problema**: Anidamiento profundo
**Solución**: Retornar temprano

```python
# Antes
def process(user):
    if user:
        if user.active:
            if user.has_permission():
                return do_something(user)
    return None

# Después
def process(user):
    if not user: return None
    if not user.active: return None
    if not user.has_permission(): return None
    return do_something(user)
```

## 2. Extract Function

**Problema**: Función hace múltiples cosas
**Solución**: Dividir en funciones más pequeñas

```python
# Antes
def process_order(order):
    # validar
    # calcular totales
    # aplicar descuentos
    # guardar
    # enviar email
    pass

# Después
def process_order(order):
    validated = validate(order)
    totals = calculate_totals(validated)
    final_order = apply_discounts(totals)
    saved = save(final_order)
    send_confirmation(saved)
```

## 3. Replace Parameter with Object

**Problema**: Múltiples parámetros
**Solución**: Agrupar en objeto/struct

```python
# Antes
def create_user(name, email, phone, address, city, country, postal_code):
    pass

# Después
@dataclass
class UserData:
    name: str
    email: str
    phone: str
    address: Address

def create_user(data: UserData):
    pass
```

## 4. Eliminar Abstracción Prematura

**Problema**: Interface/clase abstracta con 1 implementación
**Solución**: Eliminar hasta que se necesite

```python
# Antes
class UserRepository(ABC):
    @abstractmethod
    def find(id): pass

class SqlUserRepository(UserRepository):
    def find(id): ...

# Después
class UserRepository:
    def find(id): ...
```

## 5. Consolidate Conditional

**Problema**: Condicionales similares en múltiples lugares
**Solución**: Extraer a una función

```python
# Antes
if user.age >= 18 and user.verified and not user.banned:
    # ...
# ... más tarde
if user.age >= 18 and user.verified and not user.banned:
    # ...

# Después
def can_access_premium(user):
    return user.age >= 18 and user.verified and not user.banned

if can_access_premium(user):
    # ...
```

## 6. Replace Nested Conditional with Guard Clauses

**Problema**: If-else profundo
**Solución**: Guard clauses con returns tempranos

```typescript
// Antes
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

// Después
function getPrice(user) {
    const BASE_PRICE = 100;

    if (!user.premium && user.years <= 2) return BASE_PRICE;
    if (!user.premium) return BASE_PRICE * 0.9;
    if (user.years > 5) return BASE_PRICE * 0.7;
    return BASE_PRICE * 0.8;
}
```

## 7. Remove Assignments to Parameters

**Problema**: Reasignar parámetros
**Solución**: Usar variables locales

```python
# Antes
def process(data):
    data = data.strip()
    data = data.lower()
    return data

# Después
def process(raw_data):
    cleaned = raw_data.strip()
    normalized = cleaned.lower()
    return normalized
```

## Orden de Aplicación

1. **Primero**: YAGNI → Eliminar innecesario
2. **Segundo**: KISS → Simplificar complejo
3. **Tercero**: DRY → Centralizar duplicados

Este orden evita crear abstracciones sobre cosas que deberían eliminarse.
