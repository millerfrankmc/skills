# Ejemplos TypeScript

## Anti-Clever Code

**MAL**:
```typescript
const process = (data: any) => data?.filter((x: any) => x?.status ?? 0)?.map((x: any) => ({...x, ts: Date.now()})) ?? [];
```

**BIEN**:
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

## YAGNI - Eliminar Abstracción Innecesaria

**MAL**:
```typescript
interface IUserService {
  getUser(id: string): Promise<User>;
}
class UserService implements IUserService { } // Una sola implementación
```

**BIEN**:
```typescript
class UserService {
  async getUser(id: string): Promise<User> { /* ... */ }
}
```

---

## KISS - Guard Clauses

**BIEN** (guard clauses):
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

## DRY - Centralizar

**BIEN** (DRY con interfaz):
```typescript
interface HasName { firstName: string; lastName: string }

function formatName(person: HasName): string {
  return `${person.firstName} ${person.lastName}`.trim();
}
// Uso: formatName(user), formatName(admin), formatName(guest)
```
