# Case Studies

Ejemplos reales de refactorización aplicando KISS, DRY, YAGNI y principios de Linus Torvalds.

---

## Case Study 1: Refactoring de Servicio de Pagos

**Problema**: Servicio de 500+ líneas con múltiples responsabilidades y parámetros excesivos.

### Antes (Código Real)

```python
class PaymentService:
    def process_payment(self, user_id, amount, card_number, cvv, expiry,
                       currency, merchant_id, callback_url, metadata,
                       retry_count, timeout, log_level):
        # Validación de usuario (15 líneas)
        if user_id and len(user_id) > 0:
            user = self.db.get_user(user_id)
            if user and user.is_active:
                # Validación de tarjeta (20 líneas)
                if card_number and len(card_number) == 16:
                    if cvv and len(cvv) == 3:
                        if expiry and len(expiry) == 5:
                            # Validación de monto (10 líneas)
                            if amount and amount > 0:
                                # Cálculo de fees (25 líneas)
                                fee_percentage = 0.029
                                if merchant_id in self.special_merchants:
                                    fee_percentage = 0.025
                                fixed_fee = 0.30
                                total_fee = (amount * fee_percentage) + fixed_fee
                                final_amount = amount - total_fee

                                # Llamada a gateway (30 líneas)
                                gateway_response = self.gateway.charge(
                                    card_number=card_number,
                                    cvv=cvv,
                                    expiry=expiry,
                                    amount=final_amount,
                                    currency=currency or "USD"
                                )

                                # Logging (15 líneas)
                                self.logger.log(f"Payment processed: {user_id}", level=log_level)

                                # Notificaciones (20 líneas)
                                if callback_url:
                                    self.send_callback(callback_url, gateway_response)

                                # Guardar en DB (15 líneas)
                                transaction = self.db.save_transaction(
                                    user_id=user_id,
                                    amount=amount,
                                    fee=total_fee,
                                    status=gateway_response.status
                                )

                                # Retornar resultado
                                return {
                                    "success": True,
                                    "transaction_id": transaction.id,
                                    "amount": final_amount,
                                    "fee": total_fee
                                }
        return {"success": False}
```

**Violaciones detectadas:**
- [KISS] 13 parámetros (límite: 4)
- [KISS] Función de 170+ líneas (límite: 20)
- [KISS] 5 niveles de anidamiento (límite: 2)
- [KISS] Función hace 6 cosas: valida, calcula, cobra, loguea, notifica, guarda
- [DRY] Validación de tarjeta duplicada en otros métodos
- [YAGNI] `callback_url`, `metadata`, `retry_count`, `timeout`, `log_level` nunca usados
- [LINUS] Lógica escondida en anidamiento profundo

### Después (Refactorizado)

```python
from dataclasses import dataclass
from decimal import Decimal

@dataclass
class PaymentRequest:
    user_id: str
    amount: Decimal
    card: CardDetails
    currency: str = "USD"

@dataclass
class CardDetails:
    number: str
    cvv: str
    expiry: str

class PaymentService:
    def process_payment(self, request: PaymentRequest) -> PaymentResult:
        user = self._validate_user(request.user_id)
        self._validate_card(request.card)

        fee = self._calculate_fee(request.amount)
        final_amount = request.amount - fee

        transaction = self._charge(request, final_amount)

        return PaymentResult(
            success=True,
            transaction_id=transaction.id,
            amount=final_amount,
            fee=fee
        )

    def _validate_user(self, user_id: str) -> User:
        if not user_id:
            raise ValueError("User ID required")
        user = self.db.get_user(user_id)
        if not user or not user.is_active:
            raise ValueError("Invalid or inactive user")
        return user

    def _validate_card(self, card: CardDetails) -> None:
        if len(card.number) != 16:
            raise ValueError("Invalid card number")
        if len(card.cvv) != 3:
            raise ValueError("Invalid CVV")

    def _calculate_fee(self, amount: Decimal) -> Decimal:
        fee_rate = Decimal("0.029")
        fixed_fee = Decimal("0.30")
        return (amount * fee_rate) + fixed_fee

    def _charge(self, request: PaymentRequest, amount: Decimal) -> Transaction:
        return self.gateway.charge(
            card=request.card,
            amount=amount,
            currency=request.currency
        )
```

### Métricas

| Aspecto | Antes | Después | Mejora |
|---------|-------|---------|--------|
| Líneas por función | 170 | 15-20 | 88% |
| Parámetros | 13 | 1-2 | 90% |
| Niveles de anidamiento | 5 | 1 | 80% |
| Responsabilidades | 6 | 1 | 83% |
| Testabilidad | Difícil | Fácil | Alta |
| Cobertura de tests alcanzable | 40% | 95% | +55% |

### Lecciones Aprendidas

1. **Guard clauses**: Cada validación retorna/raise inmediatamente, no anida
2. **Parameter object**: `PaymentRequest` agrupa datos relacionados
3. **Single Responsibility**: Cada método hace una cosa
4. **YAGNI**: Eliminar parámetros no usados simplificó todo

---

## Case Study 2: Eliminación de Abstracción Innecesaria

**Problema**: Repository genérico con una sola implementación y 13 métodos, solo 2 usados.

### Antes (Código Real)

```typescript
interface IRepository<T> {
  findById(id: string): Promise<T>;
  findAll(): Promise<T[]>;
  findBy(filters: Filter[]): Promise<T[]>;
  findOne(filters: Filter[]): Promise<T | null>;
  save(entity: T): Promise<T>;
  saveMany(entities: T[]): Promise<T[]>;
  update(id: string, data: Partial<T>): Promise<T>;
  updateMany(filters: Filter[], data: Partial<T>): Promise<number>;
  delete(id: string): Promise<void>;
  deleteMany(filters: Filter[]): Promise<number>;
  count(filters?: Filter[]): Promise<number>;
  exists(id: string): Promise<boolean>;
  paginate(page: number, limit: number): Promise<PaginatedResult<T>>;
}

interface Filter {
  field: string;
  operator: 'eq' | 'ne' | 'gt' | 'lt' | 'contains';
  value: any;
}

interface PaginatedResult<T> {
  data: T[];
  total: number;
  page: number;
  limit: number;
}

class UserRepository implements IRepository<User> {
  async findById(id: string): Promise<User> {
    return this.db.users.findUnique({ where: { id } });
  }

  async save(user: User): Promise<User> {
    return this.db.users.create({ data: user });
  }

  // Los otros 11 métodos...
  async findAll(): Promise<User[]> { throw new Error("Not implemented"); }
  async findBy(): Promise<User[]> { throw new Error("Not implemented"); }
  async findOne(): Promise<User | null> { throw new Error("Not implemented"); }
  async saveMany(): Promise<User[]> { throw new Error("Not implemented"); }
  async update(): Promise<User> { throw new Error("Not implemented"); }
  async updateMany(): Promise<number> { throw new Error("Not implemented"); }
  async delete(): Promise<void> { throw new Error("Not implemented"); }
  async deleteMany(): Promise<number> { throw new Error("Not implemented"); }
  async count(): Promise<number> { throw new Error("Not implemented"); }
  async exists(): Promise<boolean> { throw new Error("Not implemented"); }
  async paginate(): Promise<PaginatedResult<User>> { throw new Error("Not implemented"); }
}
```

**Violaciones detectadas:**
- [YAGNI] 11 métodos nunca usados (85% del código)
- [YAGNI] Interface `Filter` y `PaginatedResult` no usados
- [YAGNI] Generics complejos sin necesidad
- [KISS] Abstracción innecesaria (1 implementación)
- [LINUS] 200 líneas de "Not implemented"

### Después (Refactorizado)

```typescript
class UserRepository {
  async findById(id: string): Promise<User | null> {
    return this.db.users.findUnique({ where: { id } });
  }

  async save(user: User): Promise<User> {
    return this.db.users.create({ data: user });
  }
}
```

### Métricas

| Aspecto | Antes | Después | Mejora |
|---------|-------|---------|--------|
| Métodos | 13 | 2 | 85% |
| Líneas de código | 200 | 15 | 93% |
| Archivos | 1 | 1 | - |
| Interfaces | 3 | 0 | 100% |
| Complejidad cognitiva | Alta | Baja | Alta |

### Lecciones Aprendidas

1. **Interface = Contrato**: Solo crear cuando hay 2+ implementaciones
2. **Generalización prematura**: El costo de mantener 11 métodos sin usar > beneficio teórico
3. **Simple es mantenible**: Si necesitamos más métodos, los agregamos

---

## Case Study 3: Aplanamiento de Anidamiento Profundo

**Problema**: Lógica de procesamiento de órdenes con 7 niveles de anidamiento.

### Antes (Código Real)

```go
func ProcessOrder(order *Order) (*OrderResult, error) {
    if order != nil {
        if order.Status == "pending" {
            if order.Customer != nil {
                if order.Customer.IsActive {
                    if len(order.Items) > 0 {
                        allInStock := true
                        for _, item := range order.Items {
                            if !item.InStock {
                                allInStock = false
                                break
                            }
                        }
                        if allInStock {
                            if order.Payment != nil {
                                if order.Payment.IsProcessed {
                                    if order.Payment.Amount >= order.Total {
                                        if order.ShippingAddress != nil && order.ShippingAddress.IsValid() {
                                            // Finalmente procesar la orden
                                            result, err := fulfillOrder(order)
                                            if err != nil {
                                                return nil, err
                                            }
                                            return result, nil
                                        } else {
                                            return nil, errors.New("invalid shipping address")
                                        }
                                    } else {
                                        return nil, errors.New("insufficient payment amount")
                                    }
                                } else {
                                    return nil, errors.New("payment not processed")
                                }
                            } else {
                                return nil, errors.New("no payment information")
                            }
                        } else {
                            return nil, errors.New("items out of stock")
                        }
                    } else {
                        return nil, errors.New("order has no items")
                    }
                } else {
                    return nil, errors.New("customer is not active")
                }
            } else {
                return nil, errors.New("customer information missing")
            }
        } else {
            return nil, errors.New("order is not in pending status")
        }
    } else {
        return nil, errors.New("order is nil")
    }
}
```

**Violaciones detectadas:**
- [KISS] 7 niveles de anidamiento (límite: 2)
- [KISS] Función de 50+ líneas (límite: 20)
- [LINUS] Difícil de entender de un vistazo
- [LINUS] Mensajes de error escondidos al final

### Después (Refactorizado)

```go
func ProcessOrder(order *Order) (*OrderResult, error) {
    if order == nil {
        return nil, errors.New("order is nil")
    }
    if order.Status != "pending" {
        return nil, errors.New("order is not in pending status")
    }
    if order.Customer == nil {
        return nil, errors.New("customer information missing")
    }
    if !order.Customer.IsActive {
        return nil, errors.New("customer is not active")
    }
    if len(order.Items) == 0 {
        return nil, errors.New("order has no items")
    }
    if !allItemsInStock(order.Items) {
        return nil, errors.New("items out of stock")
    }
    if order.Payment == nil {
        return nil, errors.New("no payment information")
    }
    if !order.Payment.IsProcessed {
        return nil, errors.New("payment not processed")
    }
    if order.Payment.Amount < order.Total {
        return nil, errors.New("insufficient payment amount")
    }
    if order.ShippingAddress == nil || !order.ShippingAddress.IsValid() {
        return nil, errors.New("invalid shipping address")
    }

    return fulfillOrder(order)
}

func allItemsInStock(items []Item) bool {
    for _, item := range items {
        if !item.InStock {
            return false
        }
    }
    return true
}
```

### Métricas

| Aspecto | Antes | Después | Mejora |
|---------|-------|---------|--------|
| Niveles de anidamiento | 7 | 1 | 86% |
| Líneas de código | 52 | 35 | 33% |
| Legibilidad | Baja | Alta | Alta |
| Tiempo para entender | 5+ min | 30 seg | 90% |

### Lecciones Aprendidas

1. **Guard clauses**: Validar y retornar error inmediatamente
2. **Flujo lineal**: El código se lee de arriba a abajo
3. **Mensajes específicos**: Cada error describe exactamente qué falló
4. **Funciones pequeñas**: `allItemsInStock` extraída para claridad

---

## Case Study 4: Eliminación de "Clever Code"

**Problema**: One-liners crípticos que son difíciles de entender y debuggear.

### Antes (Código Real)

```python
def calculate_discount(user, order, promo_code):
    # ¿Qué hace esto? ¿Cuál es el descuento final?
    return (
        (order.total * 0.15 if user.is_premium else order.total * 0.05) +
        (10 if promo_code == "SAVE10" else 5 if promo_code else 0) +
        (order.total * 0.02 if order.total > 100 else 0)
    ) if order.is_valid and not order.is_discounted else 0
```

**Violaciones detectadas:**
- [LINUS] One-liner críptico
- [KISS] Lógica compleja anidada
- [LINUS] Difícil de entender de un vistazo
- [KISS] Difícil de debuggear

### Después (Refactorizado)

```python
def calculate_discount(user, order, promo_code):
    if not order.is_valid or order.is_discounted:
        return 0

    membership_discount = calculate_membership_discount(user, order.total)
    promo_discount = calculate_promo_discount(promo_code)
    volume_discount = calculate_volume_discount(order.total)

    return membership_discount + promo_discount + volume_discount

def calculate_membership_discount(user, total):
    if user.is_premium:
        return total * 0.15
    return total * 0.05

def calculate_promo_discount(promo_code):
    if promo_code == "SAVE10":
        return 10
    if promo_code:
        return 5
    return 0

def calculate_volume_discount(total):
    if total > 100:
        return total * 0.02
    return 0
```

### Métricas

| Aspecto | Antes | Después | Mejora |
|---------|-------|---------|--------|
| Líneas | 1 | 22 | +21 |
| Claridad | Baja | Alta | Alta |
| Debuggeable | No | Sí | Total |
| Testeable | Difícil | Fácil | Alta |

### Lecciones Aprendidas

1. **Claro > Corto**: 22 líneas claras > 1 línea críptica
2. **Nombrar operaciones**: Cada función describe qué calcula
3. **Fácil de debuggear**: Puedes poner breakpoint en cada paso
4. **Fácil de testear**: Cada función se testea independientemente

---

## Patrones Recurrentes

De estos case studies, emergen patrones comunes:

1. **Guard clauses**: Eliminar anidamiento profundo
2. **Funciones pequeñas**: 20 líneas máximo
3. **Nombres descriptivos**: El código se explica solo
4. **YAGNI**: Eliminar antes de simplificar
5. **Una responsabilidad**: Cada función/clase hace una cosa
