# Case Studies

## Case Study 1: Refactoring de Servicio de Pagos

**Problema**: Servicio de 500+ líneas con múltiples responsabilidades

**Antes** (violaciones detectadas):
```python
class PaymentService:
    def process_payment(self, user_id, amount, card_number, cvv, expiry, currency, merchant_id, callback_url, metadata):
        # 50 líneas de validación
        # 30 líneas de cálculo de fees
        # 40 líneas de llamada a gateway
        # 25 líneas de logging
        # 35 líneas de notificaciones
        # Total: 180 líneas en una función
        pass
```

**Violaciones**:
- [KISS] 9 parámetros, 180 líneas
- [KISS] Una función hace 5 cosas
- [DRY] Validación duplicada en otros métodos
- [YAGNI] metadata y callback_url nunca se usan

**Después** (aplicando principios):
```python
class PaymentService:
    def process_payment(self, request: PaymentRequest) -> PaymentResult:
        validated = self._validate(request)
        charged = self._charge(validated)
        return self._build_result(charged)

    def _validate(self, request): ...
    def _charge(self, validated): ...
    def _build_result(self, charged): ...

@dataclass
class PaymentRequest:
    user_id: str
    amount: Decimal
    card: CardDetails
    currency: str = "USD"
```

**Métricas**:
| Aspecto | Antes | Después |
|--------|-------|---------|
| Líneas por función | 180 | 15-20 |
| Parámetros | 9 | 1 (objeto) |
| Responsabilidades | 5 | 1 |
| Testabilidad | Difícil | Fácil |

---

## Case Study 2: Eliminación de Abstracción Innecesaria

**Problema**: Repository genérico con una sola implementación

**Antes** (violaciones detectadas):
```typescript
interface IRepository<T> {
  findById(id: string): Promise<T>;
  findAll(): Promise<T[]>;
  save(entity: T): Promise<T>;
  delete(id: string): Promise<void>;
  // ... 10 más métodos
}

class UserRepository implements IRepository<User> {
  // Solo usa findById y save
  // Los otros 12 métodos lanzan "Not implemented"
}
```

**Violaciones**:
- [YAGNI] 12 métodos nunca usados
- [YAGNI] Interface con 1 implementación
- [KISS] Abstracción innecesaria

**Después**:
```typescript
class UserRepository {
  async findById(id: string): Promise<User | null> { /* ... */ }
  async save(user: User): Promise<User> { /* ... */ }
}
```

**Métricas**:
| Aspecto | Antes | Después |
|--------|-------|---------|
| Métodos | 13 | 2 |
| Archivos | 2 (interface + impl) | 1 |
| Líneas | 80 | 20 |

---

## Case Study 3: Aplanamiento de Anidamiento

**Problema**: Lógica de negocio profundamente anidada

**Antes** (violaciones detectadas):
```go
func ProcessOrder(order *Order) error {
    if order != nil {
        if order.IsValid() {
            if order.HasItems() {
                if order.Payment != nil {
                    if order.Payment.IsProcessed() {
                        if order.Customer.IsActive() {
                            if order.Inventory.Available() {
                                return fulfillOrder(order)
                            }
                        }
                    }
                }
            }
        }
    }
    return errors.New("invalid order")
}
```

**Violaciones**:
- [KISS] 7 niveles de anidamiento
- [LINUS] Difícil de entender de un vistazo

**Después** (guard clauses):
```go
func ProcessOrder(order *Order) error {
    if order == nil { return errors.New("nil order") }
    if !order.IsValid() { return errors.New("invalid order") }
    if !order.HasItems() { return errors.New("no items") }
    if order.Payment == nil { return errors.New("no payment") }
    if !order.Payment.IsProcessed() { return errors.New("payment not processed") }
    if !order.Customer.IsActive() { return errors.New("inactive customer") }
    if !order.Inventory.Available() { return errors.New("inventory unavailable") }

    return fulfillOrder(order)
}
```

**Métricas**:
| Aspecto | Antes | Después |
|--------|-------|---------|
| Niveles anidamiento | 7 | 1 |
| Líneas | 15 | 9 |
| Legibilidad | Baja | Alta |
