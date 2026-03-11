# Anti-Patrones Comunes

Catálogo de problemas frecuentes y sus soluciones inmediatas.

## Índice

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

**Señales:** Múltiples niveles de if/else anidados (>2 niveles)

**Problema:** Código imposible de seguir, lógica escondida

**Antes:**
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

**Después:**
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

**Señales:** Función hace 3+ cosas diferentes, >20 líneas

**Problema:** Difícil de testear, reusar, y entender

**Antes:**
```typescript
function processUser(data) {
  // Validar (10 líneas)
  // Transformar (15 líneas)
  // Guardar en DB (10 líneas)
  // Enviar email (10 líneas)
  // Loggear (5 líneas)
}
```

**Después:**
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

**Señales:** Función con 5+ parámetros

**Problema:** Difícil de llamar, fácil de confundir orden

**Antes:**
```go
func CreateUser(name, email, phone, address, city, country, zip string, active bool, premium bool) error
```

**Después:**
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

**Señales:** Interface con 1 implementación

**Problema:** Complejidad innecesaria, código más difícil de navegar

**Antes:**
```typescript
interface IUserRepository {
  findById(id: string): Promise<User>;
  save(user: User): Promise<User>;
}

class UserRepository implements IUserRepository {
  // única implementación
}
```

**Después:**
```typescript
class UserRepository {
  async findById(id: string): Promise<User> { /* ... */ }
  async save(user: User): Promise<User> { /* ... */ }
}
```

---

## Abstraction Addiction

**Señales:** Fábricas de fábricas, servicios que solo delegan

**Problema:** Indirección sin valor, stack traces imposibles

**Antes:**
```java
UserService -> UserRepositoryFactory -> UserRepositoryImpl
```

**Después:**
```java
UserService -> UserRepository
```

---

## Future Proofing

**Señales:** Código "por si acaso lo necesitamos"

**Problema:** 80% nunca se usa, complejidad real ahora

**Antes:**
```python
class UserService:
    def create_user(self, data):
        # Soporte para múltiples tipos de usuario (solo usamos uno)
        # Soporte para validación pluguable (solo usamos una)
        # Soporte para múltiples backends (solo usamos uno)
        pass
```

**Después:**
```python
def create_user(data):
    if not is_valid(data): raise ValueError("Invalid")
    return db.users.insert(data)
```

---

## Clever Code

**Señales:** One-liners, operadores crípticos, "mira qué corto"

**Problema:** Difícil de entender, debugging difícil

**Antes:**
```python
result = data and [x for x in data if x.valid] or default
```

**Después:**
```python
if not data:
    return default

valid_items = [x for x in data if x.valid]
return valid_items if valid_items else default
```

---

## Comment Cancer

**Señales:** Comentarios explicando "qué" hace el código

**Problema:** Comentarios mienten, código no

**Antes:**
```python
# Incrementa el contador
i = i + 1

# Valida el email
if "@" in email and "." in email:
    # Es válido
    valid = True
```

**Después:**
```python
counter += 1

if is_valid_email(email):
    valid = True
```

---

## Resumen de Señales de Alerta

| Anti-Patrón | Señal Principal | Acción Inmediata |
|-------------|-----------------|------------------|
| Pyramid of Doom | >2 niveles de if | Guard clauses |
| God Function | >20 líneas | Extraer funciones |
| Parameter Explosion | >4 parámetros | Objeto config |
| Interface Pollution | 1 implementación | Eliminar interface |
| Abstraction Addiction | Fábricas de fábricas | Simplificar |
| Future Proofing | Código no usado | Eliminar |
| Clever Code | One-liners | Expandir |
| Comment Cancer | // qué hace | Renombrar |
