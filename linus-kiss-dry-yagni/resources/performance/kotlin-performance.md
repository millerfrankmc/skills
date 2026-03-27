# Performance Specific - Kotlin

> Guide for performance optimizations in Kotlin code (JVM/Android). NEVER sacrifice obvious performance for "cleaner code".

---

## 1. Coroutines vs Threads

### The Problem

```kotlin
// ❌ INEFFICIENT - Creating threads manually
fun processConcurrently(items: List<Item>) {
    val threads = items.map { item ->
        Thread { process(item) }.apply { start() }
    }
    threads.forEach { it.join() } // OS thread overhead
}

// ❌ INEFFICIENT - Blocking calls in coroutines
suspend fun fetchData() {
    withContext(Dispatchers.IO) {
        // Blocks the pool thread
        Thread.sleep(1000)
    }
}
```

### The Solution

```kotlin
// ✅ CORRECT - Coroutines with appropriate dispatchers
import kotlinx.coroutines.*

suspend fun processConcurrently(items: List<Item>) {
    // Launch all coroutines concurrently
    val jobs = items.map { item ->
        async(Dispatchers.Default) {
            process(item)
        }
    }

    // Wait for all results
    val results = jobs.awaitAll()
}

// ✅ CORRECT - Non-blocking suspend functions
suspend fun fetchData(): Data {
    // delay is suspend, doesn't block thread
    delay(1000)
    return Data()
}

// ✅ CORRECT - Flow for data streams
import kotlinx.coroutines.flow.*

fun fetchPaginatedData(): Flow<Page> = flow {
    var page = 0
    do {
        val data = fetchPage(page++) // Suspend function
        emit(data)
    } while (data.hasMore)
}.buffer(10) // Buffer for backpressure

// Consume
fetchPaginatedData()
    .map { process(it) }
    .collect { save(it) }

// ✅ CORRECT - Structured concurrency
suspend fun processWithTimeout(items: List<Item>) = coroutineScope {
    val deferred = items.map { item ->
        async {
            withTimeout(5000) {
                process(item)
            }
        }
    }

    deferred.awaitAll()
}
```

---

## 2. Lazy Initialization

```kotlin
// ✅ CORRECT - Lazy delegate
class DatabaseConnection {
    // Only initializes first time accessed
    val connection: Connection by lazy {
        println("Creating connection...")
        createExpensiveConnection()
    }

    // Lazy thread-safe (default)
    val cache: Cache by lazy(LazyThreadSafetyMode.SYNCHRONIZED) {
        Cache()
    }
}

// ✅ CORRECT - Lazy in class properties
class ConfigLoader {
    // Doesn't load until needed
    val config: Config by lazy { loadFromDisk() }

    fun getSetting(key: String): String? {
        return config[key] // First call loads config
    }
}

// ✅ CORRECT - Lateinit for injection
class UserService {
    lateinit var repository: UserRepository

    fun init(repo: UserRepository) {
        this.repository = repo
    }

    fun getUser(id: String): User {
        check(::repository.isInitialized) { "Repository not initialized" }
        return repository.findById(id)
    }
}
```

---

## 3. Inline Functions

```kotlin
// ❌ INEFFICIENT - Lambda creates anonymous object
fun measureTime(block: () -> Unit): Long {
    val start = System.currentTimeMillis()
    block()
    return System.currentTimeMillis() - start
}

// Each call creates: object : Function0<Unit> { ... }
measureTime { doSomething() }

// ✅ CORRECT - inline eliminates lambda overhead
inline fun measureTime(block: () -> Unit): Long {
    val start = System.currentTimeMillis()
    block()
    return System.currentTimeMillis() - start
}

// Compiler inlines body, no lambda object
measureTime { doSomething() }
// Compiles to:
// val start = System.currentTimeMillis()
// doSomething()
// val result = System.currentTimeMillis() - start

// ✅ CORRECT - inline with reified type parameters
inline fun <reified T> Gson.fromJson(json: String): T {
    return fromJson(json, T::class.java)
}

// Usage without Class<T>
val user: User = gson.fromJson(jsonString)

// ✅ CORRECT - inline for higher-order functions
inline fun <T> T.applyIf(condition: Boolean, block: T.() -> Unit): T {
    if (condition) block()
    return this
}
```

---

## 4. Sequence vs Collection

```kotlin
// ❌ INEFFICIENT - Multiple intermediate collections
fun processLargeList(numbers: List<Int>): List<Int> {
    return numbers
        .map { it * 2 }        // New list
        .filter { it > 10 }    // New list
        .take(10)              // New list
}

// ✅ CORRECT - Sequence for lazy evaluation
fun processLargeList(numbers: List<Int>): List<Int> {
    return numbers
        .asSequence()
        .map { it * 2 }
        .filter { it > 10 }
        .take(10)
        .toList() // Only one final list
}

// ✅ CORRECT - generateSequence for infinite sequences
val fibonacci = generateSequence(Pair(0, 1)) { (a, b) ->
    Pair(b, a + b)
}.map { it.first }

// Take only first 10
val first10 = fibonacci.take(10).toList()

// ✅ CORRECT - sequence builder for complex cases
fun fetchPaginated(): Sequence<Data> = sequence {
    var page = 0
    do {
        val response = fetchPage(page++)
        yieldAll(response.items)
    } while (response.hasMore)
}

// Use without loading everything in memory
fetchPaginated()
    .filter { it.isValid }
    .take(100)
    .forEach { process(it) }
```

---

## 5. JVM Tuning (GC, Heap)

```kotlin
// ✅ CORRECT - Annotations for JVM optimization

// Avoid boxing in primitive arrays
val numbers: IntArray = intArrayOf(1, 2, 3) // int[] in JVM
// vs
val boxed: Array<Int> = arrayOf(1, 2, 3) // Integer[]

// ✅ CORRECT - JvmOverloads for less code
@JvmOverloads
fun greet(name: String, greeting: String = "Hello") {
    println("$greeting, $name!")
}
// Generates:
// greet(String name, String greeting)
// greet(String name)

// ✅ CORRECT - JvmStatic for efficient static access
object Config {
    @JvmStatic
    fun getValue(key: String): String = //...
}

// ✅ CORRECT - JvmInline for value classes
@JvmInline
value class UserId(val value: String)

// At runtime it's just String, no wrapper object
fun findUser(id: UserId): User

// ✅ CORRECT - const for compile-time constants
const val MAX_RETRIES = 3 // Inline in bytecode
// vs
val maxRetries = 3 // Property with getter
```

---

## 6. Flow for Reactive Streams

```kotlin
import kotlinx.coroutines.flow.*

// ✅ CORRECT - Flow for data streams
class UserRepository {
    fun getUsersFlow(): Flow<User> = flow {
        // Emit one by one
        database.getUsers().forEach { user ->
            emit(user)
        }
    }
}

// ✅ CORRECT - Flow operators
userRepository.getUsersFlow()
    .filter { it.isActive }
    .map { it.toDto() }
    .buffer(100) // Buffer between producer and consumer
    .conflate() // Skip elements if consumer is slow
    .collect { display(it) }

// ✅ CORRECT - StateFlow for shared state
class ViewModel {
    private val _uiState = MutableStateFlow(UiState())
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()

    fun updateState(newState: UiState) {
        _uiState.value = newState
    }
}

// ✅ CORRECT - SharedFlow for events
class EventBus {
    private val _events = MutableSharedFlow<Event>(
        extraBufferCapacity = 10,
        onBufferOverflow = BufferOverflow.DROP_OLDEST
    )
    val events = _events.asSharedFlow()

    fun emit(event: Event) {
        _events.tryEmit(event)
    }
}

// ✅ CORRECT - combine for multiple flows
combine(
    userFlow,
    settingsFlow,
    networkStatusFlow
) { user, settings, network ->
    UiState(user, settings, network)
}.collect { updateUi(it) }
```

---

## 7. Android: ViewHolder, RecyclerView Optimization

```kotlin
// ✅ CORRECT - ViewHolder pattern
class MyAdapter : RecyclerView.Adapter<MyAdapter.ViewHolder>() {
    // ViewHolder reuses views
    class ViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
        val nameText: TextView = itemView.findViewById(R.id.name)
        val imageView: ImageView = itemView.findViewById(R.id.image)
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ViewHolder {
        val view = LayoutInflater.from(parent.context)
            .inflate(R.layout.item_layout, parent, false)
        return ViewHolder(view)
    }

    override fun onBindViewHolder(holder: ViewHolder, position: Int) {
        val item = getItem(position)
        holder.nameText.text = item.name
        // Load image with Glide/Picasso (with caching)
        Glide.with(holder.itemView).load(item.imageUrl).into(holder.imageView)
    }
}

// ✅ CORRECT - ListAdapter for efficient diffing
class UserAdapter : ListAdapter<User, UserAdapter.ViewHolder>(
    DiffCallback()
) {
    class DiffCallback : DiffUtil.ItemCallback<User>() {
        override fun areItemsTheSame(old: User, new: User) =
            old.id == new.id

        override fun areContentsTheSame(old: User, new: User) =
            old == new
    }
    // ListAdapter calculates diff in background and animates changes
}

// ✅ CORRECT - ViewBinding for efficient access
class ViewHolder(binding: ItemLayoutBinding) :
    RecyclerView.ViewHolder(binding.root) {

    fun bind(item: Item) {
        binding.name.text = item.name
        binding.description.text = item.description
    }
}

// ✅ CORRECT - RecyclerView.RecycledViewPool for multiple RVs
val pool = RecyclerView.RecycledViewPool()
recyclerView1.setRecycledViewPool(pool)
recyclerView2.setRecycledViewPool(pool)
```

---

## Kotlin Specific Checklist

### Coroutines
- [ ] async/await for parallelism
- [ ] Flow for data streams
- [ ] Appropriate dispatchers (Default, IO, Main)
- [ ] Structured concurrency
- [ ] withTimeout for operations with limits

### Collections
- [ ] Sequence for operation chains
- [ ] IntArray, LongArray instead of Array<Int>
- [ ] generateSequence for infinite sequences

### Optimization
- [ ] inline for higher-order functions
- [ ] lazy for deferred initialization
- [ ] const for compile-time constants
- [ ] @JvmInline for value classes

### JVM
- [ ] Appropriate heap configuration
- [ ] GC tuning according to workload
- [ ] Avoid unnecessary boxing

### Android
- [ ] ViewHolder for RecyclerView
- [ ] ListAdapter with DiffUtil
- [ ] ViewBinding instead of findViewById
- [ ] Shared RecycledViewPool
- [ ] Glide/Picasso with caching

### Memory
- [ ] WeakReference for caches
- [ ] onCleared() in ViewModels
- [ ] Cancel coroutines in onDestroy

---

## References

- [Kotlin Coroutines Guide](https://kotlinlang.org/docs/coroutines-guide.html)
- [Kotlin Flow](https://kotlinlang.org/docs/flow.html)
- [Android Performance](https://developer.android.com/topic/performance)
- [JVM Performance](https://kotlinlang.org/docs/java-to-kotlin-interop.html)
