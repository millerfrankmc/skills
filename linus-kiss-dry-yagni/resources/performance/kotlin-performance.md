# Rendimiento Específico - Kotlin

> Guía de optimizaciones de rendimiento para código Kotlin (JVM/Android). NUNCA sacrificar rendimiento obvio por "código más limpio".

---

## 1. Coroutines vs Threads

### El Problema

```kotlin
// ❌ INEFICIENTE - Crear threads manualmente
fun processConcurrently(items: List<Item>) {
    val threads = items.map { item ->
        Thread { process(item) }.apply { start() }
    }
    threads.forEach { it.join() } // Overhead de threads del OS
}

// ❌ INEFICIENTE - Blocking calls en coroutines
suspend fun fetchData() {
    withContext(Dispatchers.IO) {
        // Bloquea el thread del pool
        Thread.sleep(1000)
    }
}
```

### La Solución

```kotlin
// ✅ CORRECTO - Coroutines con dispatchers apropiados
import kotlinx.coroutines.*

suspend fun processConcurrently(items: List<Item>) {
    // Lanzar todas las coroutines concurrentemente
    val jobs = items.map { item ->
        async(Dispatchers.Default) {
            process(item)
        }
    }

    // Esperar todos los resultados
    val results = jobs.awaitAll()
}

// ✅ CORRECTO - Suspend functions no bloqueantes
suspend fun fetchData(): Data {
    // delay es suspend, no bloquea el thread
    delay(1000)
    return Data()
}

// ✅ CORRECTO - Flow para streams de datos
import kotlinx.coroutines.flow.*

fun fetchPaginatedData(): Flow<Page> = flow {
    var page = 0
    do {
        val data = fetchPage(page++) // Suspend function
        emit(data)
    } while (data.hasMore)
}.buffer(10) // Buffer para backpressure

// Consumir
fetchPaginatedData()
    .map { process(it) }
    .collect { save(it) }

// ✅ CORRECTO - Structured concurrency
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
// ✅ CORRECTO - Delegado lazy
class DatabaseConnection {
    // Se inicializa solo la primera vez que se accede
    val connection: Connection by lazy {
        println("Creando conexión...")
        createExpensiveConnection()
    }

    // Lazy thread-safe (por defecto)
    val cache: Cache by lazy(LazyThreadSafetyMode.SYNCHRONIZED) {
        Cache()
    }
}

// ✅ CORRECTO - Lazy en propiedades de clase
class ConfigLoader {
    // No carga hasta que se necesite
    val config: Config by lazy { loadFromDisk() }

    fun getSetting(key: String): String? {
        return config[key] // Primera llamada carga el config
    }
}

// ✅ CORRECTO - Lateinit para inyección
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
// ❌ INEFICIENTE - Lambda crea objeto anónimo
fun measureTime(block: () -> Unit): Long {
    val start = System.currentTimeMillis()
    block()
    return System.currentTimeMillis() - start
}

// Cada llamada crea: object : Function0<Unit> { ... }
measureTime { doSomething() }

// ✅ CORRECTO - inline elimina overhead de lambda
inline fun measureTime(block: () -> Unit): Long {
    val start = System.currentTimeMillis()
    block()
    return System.currentTimeMillis() - start
}

// El compilador in-linea el cuerpo, sin objeto lambda
measureTime { doSomething() }
// Compila a:
// val start = System.currentTimeMillis()
// doSomething()
// val result = System.currentTimeMillis() - start

// ✅ CORRECTO - inline con reified type parameters
inline fun <reified T> Gson.fromJson(json: String): T {
    return fromJson(json, T::class.java)
}

// Uso sin Class<T>
val user: User = gson.fromJson(jsonString)

// ✅ CORRECTO - inline para higher-order functions
inline fun <T> T.applyIf(condition: Boolean, block: T.() -> Unit): T {
    if (condition) block()
    return this
}
```

---

## 4. Sequence vs Collection

```kotlin
// ❌ INEFICIENTE - Múltiples colecciones intermedias
fun processLargeList(numbers: List<Int>): List<Int> {
    return numbers
        .map { it * 2 }        // Nueva lista
        .filter { it > 10 }    // Nueva lista
        .take(10)              // Nueva lista
}

// ✅ CORRECTO - Sequence para lazy evaluation
fun processLargeList(numbers: List<Int>): List<Int> {
    return numbers
        .asSequence()
        .map { it * 2 }
        .filter { it > 10 }
        .take(10)
        .toList() // Solo una lista final
}

// ✅ CORRECTO - generateSequence para secuencias infinitas
val fibonacci = generateSequence(Pair(0, 1)) { (a, b) ->
    Pair(b, a + b)
}.map { it.first }

// Tomar solo los primeros 10
val first10 = fibonacci.take(10).toList()

// ✅ CORRECTO - sequence builder para casos complejos
fun fetchPaginated(): Sequence<Data> = sequence {
    var page = 0
    do {
        val response = fetchPage(page++)
        yieldAll(response.items)
    } while (response.hasMore)
}

// Usar sin cargar todo en memoria
fetchPaginated()
    .filter { it.isValid }
    .take(100)
    .forEach { process(it) }
```

---

## 5. JVM Tuning (GC, Heap)

```kotlin
// ✅ CORRECTO - Anotaciones para optimización JVM

// Evitar boxing en arrays primitivos
val numbers: IntArray = intArrayOf(1, 2, 3) // int[] en JVM
// vs
val boxed: Array<Int> = arrayOf(1, 2, 3) // Integer[]

// ✅ CORRECTO - JvmOverloads para menos código
@JvmOverloads
fun greet(name: String, greeting: String = "Hello") {
    println("$greeting, $name!")
}
// Genera:
// greet(String name, String greeting)
// greet(String name)

// ✅ CORRECTO - JvmStatic para acceso estático eficiente
object Config {
    @JvmStatic
    fun getValue(key: String): String = //...
}

// ✅ CORRECTO - JvmInline para value classes
@JvmInline
value class UserId(val value: String)

// En tiempo de ejecución es solo String, no objeto wrapper
fun findUser(id: UserId): User

// ✅ CORRECTO - const para compile-time constants
const val MAX_RETRIES = 3 // Inline en bytecode
// vs
val maxRetries = 3 // Property con getter
```

---

## 6. Flow para Streams Reactivos

```kotlin
import kotlinx.coroutines.flow.*

// ✅ CORRECTO - Flow para streams de datos
class UserRepository {
    fun getUsersFlow(): Flow<User> = flow {
        // Emitir uno por uno
        database.getUsers().forEach { user ->
            emit(user)
        }
    }
}

// ✅ CORRECTO - Operadores de Flow
userRepository.getUsersFlow()
    .filter { it.isActive }
    .map { it.toDto() }
    .buffer(100) // Buffer entre productor y consumidor
    .conflate() // Saltar elementos si el consumidor es lento
    .collect { display(it) }

// ✅ CORRECTO - StateFlow para estado compartido
class ViewModel {
    private val _uiState = MutableStateFlow(UiState())
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()

    fun updateState(newState: UiState) {
        _uiState.value = newState
    }
}

// ✅ CORRECTO - SharedFlow para eventos
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

// ✅ CORRECTO - combine para múltiples flows
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
// ✅ CORRECTO - ViewHolder pattern
class MyAdapter : RecyclerView.Adapter<MyAdapter.ViewHolder>() {
    // ViewHolder reutiliza views
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
        // Cargar imagen con Glide/Picasso (con caching)
        Glide.with(holder.itemView).load(item.imageUrl).into(holder.imageView)
    }
}

// ✅ CORRECTO - ListAdapter para diffing eficiente
class UserAdapter : ListAdapter<User, UserAdapter.ViewHolder>(
    DiffCallback()
) {
    class DiffCallback : DiffUtil.ItemCallback<User>() {
        override fun areItemsTheSame(old: User, new: User) =
            old.id == new.id

        override fun areContentsTheSame(old: User, new: User) =
            old == new
    }
    // ListAdapter calcula diff en background y anima cambios
}

// ✅ CORRECTO - ViewBinding para acceso eficiente
class ViewHolder(binding: ItemLayoutBinding) :
    RecyclerView.ViewHolder(binding.root) {

    fun bind(item: Item) {
        binding.name.text = item.name
        binding.description.text = item.description
    }
}

// ✅ CORRECTO - RecyclerView.RecycledViewPool para múltiples RVs
val pool = RecyclerView.RecycledViewPool()
recyclerView1.setRecycledViewPool(pool)
recyclerView2.setRecycledViewPool(pool)
```

---

## Checklist Kotlin Específico

### Coroutines
- [ ] async/await para paralelismo
- [ ] Flow para streams de datos
- [ ] Dispatchers apropiados (Default, IO, Main)
- [ ] Structured concurrency
- [ ] withTimeout para operaciones con límite

### Collections
- [ ] Sequence para cadenas de operaciones
- [ ] IntArray, LongArray en lugar de Array<Int>
- [ ] generateSequence para secuencias infinitas

### Optimización
- [ ] inline para higher-order functions
- [ ] lazy para inicialización diferida
- [ ] const para compile-time constants
- [ ] @JvmInline para value classes

### JVM
- [ ] Configuración de heap apropiada
- [ ] GC tuning según workload
- [ ] Evitar boxing innecesario

### Android
- [ ] ViewHolder para RecyclerView
- [ ] ListAdapter con DiffUtil
- [ ] ViewBinding en lugar de findViewById
- [ ] RecycledViewPool compartido
- [ ] Glide/Picasso con caching

### Memoria
- [ ] WeakReference para cachés
- [ ] onCleared() en ViewModels
- [ ] Cancelar coroutines en onDestroy

---

## Referencias

- [Kotlin Coroutines Guide](https://kotlinlang.org/docs/coroutines-guide.html)
- [Kotlin Flow](https://kotlinlang.org/docs/flow.html)
- [Android Performance](https://developer.android.com/topic/performance)
- [JVM Performance](https://kotlinlang.org/docs/java-to-kotlin-interop.html)
