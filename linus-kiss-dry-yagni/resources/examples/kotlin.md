# Kotlin Examples

## Anti-Clever Code

**BAD**:
```kotlin
fun process(items: List<Any?>) = items.filterNotNull().filter { (it as? String)?.isNotBlank() == true }.map { it.toString().uppercase() }.takeIf { it.isNotEmpty() } ?: emptyList()
```

**GOOD**:
```kotlin
fun processItems(items: List<Any?>): List<String> {
    val nonBlankStrings = items
        .filterNotNull()
        .mapNotNull { it as? String }
        .filter { it.isNotBlank() }
    return nonBlankStrings.map { it.uppercase() }
}
```

---

## YAGNI - Remove Unnecessary Abstraction

**BAD**:
```kotlin
interface Repository<T> { fun findById(id: String): T? }
class UserRepository : Repository<User> { } // Single implementation
```

**GOOD**:
```kotlin
class UserRepository { fun findById(id: String): User? { /* ... */ } }
```

---

## KISS - Guard Clauses

**GOOD** (guard clauses):
```kotlin
fun process(user: User?): Result {
    if (user == null) return Result.Error
    if (!user.isActive) return Result.Error
    if (!user.hasQuota()) return Result.Error
    return Result.Success(user.name)
}
```

---

## DRY - Centralize

**GOOD** (DRY):
```kotlin
fun calculateShipping(total: Double): Double {
    return if (total > 100) 0.0 else 9.99
}
// Usage: calculateShipping(order.total), calculateShipping(cart.total)
```
