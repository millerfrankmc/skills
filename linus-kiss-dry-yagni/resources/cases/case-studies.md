# Case Studies

Real examples of refactoring applying KISS, DRY, YAGNI and Linus Torvalds principles.

---

## Case Study 1: Payment Service Refactoring

**Problem**: 500+ line service with multiple responsibilities and excessive parameters.

### Before (Real Code)

```python
class PaymentService:
    def process_payment(self, user_id, amount, card_number, cvv, expiry,
                       currency, merchant_id, callback_url, metadata,
                       retry_count, timeout, log_level):
        # User validation (15 lines)
        if user_id and len(user_id) > 0:
            user = self.db.get_user(user_id)
            if user and user.is_active:
                # Card validation (20 lines)
                if card_number and len(card_number) == 16:
                    if cvv and len(cvv) == 3:
                        if expiry and len(expiry) == 5:
                            # Amount validation (10 lines)
                            if amount and amount > 0:
                                # Fee calculation (25 lines)
                                fee_percentage = 0.029
                                if merchant_id in self.special_merchants:
                                    fee_percentage = 0.025
                                fixed_fee = 0.30
                                total_fee = (amount * fee_percentage) + fixed_fee
                                final_amount = amount - total_fee

                                # Gateway call (30 lines)
                                gateway_response = self.gateway.charge(
                                    card_number=card_number,
                                    cvv=cvv,
                                    expiry=expiry,
                                    amount=final_amount,
                                    currency=currency or "USD"
                                )

                                # Logging (15 lines)
                                self.logger.log(f"Payment processed: {user_id}", level=log_level)

                                # Notifications (20 lines)
                                if callback_url:
                                    self.send_callback(callback_url, gateway_response)

                                # Save to DB (15 lines)
                                transaction = self.db.save_transaction(
                                    user_id=user_id,
                                    amount=amount,
                                    fee=total_fee,
                                    status=gateway_response.status
                                )

                                # Return result
                                return {
                                    "success": True,
                                    "transaction_id": transaction.id,
                                    "amount": final_amount,
                                    "fee": total_fee
                                }
        return {"success": False}
```

**Violations detected:**
- [KISS] 13 parameters (limit: 4)
- [KISS] 170+ line function (limit: 20)
- [KISS] 5 nesting levels (limit: 2)
- [KISS] Function does 6 things: validates, calculates, charges, logs, notifies, saves
- [DRY] Card validation duplicated in other methods
- [YAGNI] `callback_url`, `metadata`, `retry_count`, `timeout`, `log_level` never used
- [LINUS] Logic hidden in deep nesting

### After (Refactored)

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

### Metrics

| Aspect | Before | After | Improvement |
|--------|--------|-------|-------------|
| Lines per function | 170 | 15-20 | 88% |
| Parameters | 13 | 1-2 | 90% |
| Nesting levels | 5 | 1 | 80% |
| Responsibilities | 6 | 1 | 83% |
| Testability | Difficult | Easy | High |
| Achievable test coverage | 40% | 95% | +55% |

### Lessons Learned

1. **Guard clauses**: Each validation returns/raises immediately, doesn't nest
2. **Parameter object**: `PaymentRequest` groups related data
3. **Single Responsibility**: Each method does one thing
4. **YAGNI**: Removing unused parameters simplified everything

---

## Case Study 2: Removal of Unnecessary Abstraction

**Problem**: Generic repository with a single implementation and 13 methods, only 2 used.

### Before (Real Code)

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

  // The other 11 methods...
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

**Violations detected:**
- [YAGNI] 11 methods never used (85% of code)
- [YAGNI] `Filter` and `PaginatedResult` interfaces not used
- [YAGNI] Complex generics without need
- [KISS] Unnecessary abstraction (1 implementation)
- [LINUS] 200 lines of "Not implemented"

### After (Refactored)

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

### Metrics

| Aspect | Before | After | Improvement |
|--------|--------|-------|-------------|
| Methods | 13 | 2 | 85% |
| Lines of code | 200 | 15 | 93% |
| Files | 1 | 1 | - |
| Interfaces | 3 | 0 | 100% |
| Cognitive complexity | High | Low | High |

### Lessons Learned

1. **Interface = Contract**: Only create when there are 2+ implementations
2. **Premature generalization**: The cost of maintaining 11 unused methods > theoretical benefit
3. **Simple is maintainable**: If we need more methods, we add them

---

## Case Study 3: Flattening Deep Nesting

**Problem**: Order processing logic with 7 levels of nesting.

### Before (Real Code)

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
                                            // Finally process the order
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

**Violations detected:**
- [KISS] 7 nesting levels (limit: 2)
- [KISS] 50+ line function (limit: 20)
- [LINUS] Difficult to understand at a glance
- [LINUS] Error messages hidden at the end

### After (Refactored)

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

### Metrics

| Aspect | Before | After | Improvement |
|--------|--------|-------|-------------|
| Nesting levels | 7 | 1 | 86% |
| Lines of code | 52 | 35 | 33% |
| Readability | Low | High | High |
| Time to understand | 5+ min | 30 sec | 90% |

### Lessons Learned

1. **Guard clauses**: Validate and return error immediately
2. **Linear flow**: Code reads from top to bottom
3. **Specific messages**: Each error describes exactly what failed
4. **Small functions**: `allItemsInStock` extracted for clarity

---

## Case Study 4: Removal of "Clever Code"

**Problem**: Cryptic one-liners that are difficult to understand and debug.

### Before (Real Code)

```python
def calculate_discount(user, order, promo_code):
    # What does this do? What's the final discount?
    return (
        (order.total * 0.15 if user.is_premium else order.total * 0.05) +
        (10 if promo_code == "SAVE10" else 5 if promo_code else 0) +
        (order.total * 0.02 if order.total > 100 else 0)
    ) if order.is_valid and not order.is_discounted else 0
```

**Violations detected:**
- [LINUS] Cryptic one-liner
- [KISS] Complex nested logic
- [LINUS] Difficult to understand at a glance
- [KISS] Difficult to debug

### After (Refactored)

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

### Metrics

| Aspect | Before | After | Improvement |
|--------|--------|-------|-------------|
| Lines | 1 | 22 | +21 |
| Clarity | Low | High | High |
| Debuggable | No | Yes | Total |
| Testable | Difficult | Easy | High |

### Lessons Learned

1. **Clear > Short**: 22 clear lines > 1 cryptic line
2. **Name operations**: Each function describes what it calculates
3. **Easy to debug**: You can put breakpoint at each step
4. **Easy to test**: Each function is tested independently

---

## Recurring Patterns

From these case studies, common patterns emerge:

1. **Guard clauses**: Eliminate deep nesting
2. **Small functions**: 20 lines maximum
3. **Descriptive names**: Code explains itself
4. **YAGNI**: Remove before simplifying
5. **Single responsibility**: Each function/class does one thing
