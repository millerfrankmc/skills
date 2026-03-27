# TypeScript Examples

## Anti-Clever Code

**BAD**:
```typescript
const process = (data: any) => data?.filter((x: any) => x?.status ?? 0)?.map((x: any) => ({...x, ts: Date.now()})) ?? [];
```

**GOOD**:
```typescript
interface Item { status?: number; [key: string]: unknown }
interface ProcessedItem extends Item { ts: number }

function processItems(data: Item[] | undefined): ProcessedItem[] {
    if (!data) return [];
    const activeItems = data.filter(item => item.status ?? 0);
    return activeItems.map(item => ({ ...item, ts: Date.now() }));
}
```

---

## YAGNI - Remove Unnecessary Abstraction

**BAD**:
```typescript
interface IUserService {
  getUser(id: string): Promise<User>;
}
class UserService implements IUserService { } // Single implementation
```

**GOOD**:
```typescript
class UserService {
  async getUser(id: string): Promise<User> { /* ... */ }
}
```

---

## KISS - Guard Clauses

**GOOD** (guard clauses):
```typescript
function processOrder(order: Order | null): Result {
  if (!order) return { success: false };
  if (order.items.length === 0) return { success: false };
  if (!order.payment) return { success: false };
  if (!order.payment.validated) return { success: false };
  return { success: true };
}
```

---

## DRY - Centralize

**GOOD** (DRY with interface):
```typescript
interface HasName { firstName: string; lastName: string }

function formatName(person: HasName): string {
  return `${person.firstName} ${person.lastName}`.trim();
}
// Usage: formatName(user), formatName(admin), formatName(guest)
```
