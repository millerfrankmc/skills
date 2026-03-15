# Seguridad Específica - Kotlin

> Guía de protecciones de seguridad para código Kotlin (JVM/Android). NUNCA eliminar estos patrones al aplicar KISS-DRY-YAGNI.

---

## 1. Null Safety

### El Problema
```kotlin
// ❌ Java-style null (puede causar NPE)
var name: String? = null
val length = name.length // NullPointerException!

// ❌ Forzar non-null sin verificar
val name: String = nullableName!! // Crash si es null
```

### La Solución
```kotlin
// ✅ CORRECTO - Safe call con elvis operator
val length = name?.length ?: 0

// ✅ CORRECTO - Safe call con early return
fun processUser(user: User?) {
    val name = user?.name ?: return // Early return si null
    // Procesar name
}

// ✅ CORRECTO - Smart cast después de verificación
fun processUser(user: User?) {
    if (user == null) return
    // Ahora user es non-null (smart cast)
    println(user.name)
}

// ✅ CORRECTO - let para scopes seguros
user?.let { safeUser ->
    // safeUser es non-null aquí
    process(safeUser)
}
```

---

## 2. Inyección de Dependencias Segura

### Spring Boot
```kotlin
// ✅ CORRECTO - Constructor injection (recomendado)
@Service
class UserService(
    private val userRepository: UserRepository,
    private val passwordEncoder: PasswordEncoder
) {
    fun createUser(request: CreateUserRequest): User {
        // Usar injected dependencies
    }
}

// ❌ EVITAR - Field injection
@Service
class BadUserService {
    @Autowired
    private lateinit var userRepository: UserRepository // Puede no estar inicializado
}

// ✅ CORRECTO - Validación de input
@PostMapping("/users")
fun createUser(
    @Valid @RequestBody request: CreateUserRequest
): ResponseEntity<User> {
    // Validación automática por @Valid
    return ResponseEntity.ok(userService.create(request))
}

data class CreateUserRequest(
    @field:NotBlank
    @field:Size(min = 3, max = 50)
    val username: String,

    @field:Email
    val email: String,

    @field:Size(min = 8)
    val password: String
)
```

---

## 3. SQL Injection (JPA/JDBC)

```kotlin
// ❌ PROHIBIDO - Concatenación de strings
@Query("SELECT * FROM users WHERE name = '$name'")
fun findByName(name: String): List<User>

// ✅ CORRECTO - Parámetros nombrados
@Query("SELECT u FROM User u WHERE u.name = :name")
fun findByName(@Param("name") name: String): List<User>

// ✅ CORRECTO - Métodos derivados (Spring genera query segura)
fun findByUsernameAndActive(username: String, active: Boolean): List<User>

// ✅ CORRECTO - Criteria API para queries dinámicas
fun findUsers(criteria: UserCriteria): List<User> {
    val cb = entityManager.criteriaBuilder
    val query = cb.createQuery(User::class.java)
    val root = query.from(User::class.java)

    val predicates = mutableListOf<Predicate>()

    criteria.name?.let {
        predicates.add(cb.like(root.get("name"), "%$it%"))
    }

    criteria.age?.let {
        predicates.add(cb.equal(root.get("age"), it))
    }

    query.where(*predicates.toTypedArray())
    return entityManager.createQuery(query).resultList
}

// ✅ CORRECTO - JPA Repository con JPQL seguro
interface UserRepository : JpaRepository<User, Long> {
    @Query("SELECT u FROM User u WHERE u.email = ?1")
    fun findByEmail(email: String): User?
}
```

---

## 4. Password Hashing

```kotlin
// ✅ CORRECTO - BCrypt (Spring Security)
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder

@Configuration
class SecurityConfig {
    @Bean
    fun passwordEncoder(): PasswordEncoder = BCryptPasswordEncoder(12)
}

@Service
class AuthService(
    private val passwordEncoder: PasswordEncoder
) {
    fun hashPassword(password: String): String {
        return passwordEncoder.encode(password)
    }

    fun verifyPassword(password: String, hashed: String): Boolean {
        return passwordEncoder.matches(password, hashed)
    }
}

// ✅ CORRECTO - Argon2 (más moderno)
import de.mkammerer.argon2.Argon2Factory

class Argon2PasswordHasher {
    private val argon2 = Argon2Factory.create(
        Argon2Factory.Argon2Types.ARGON2id,
        16,  // Salt length
        32   // Hash length
    )

    fun hash(password: String): String {
        return argon2.hash(2, 65536, 1, password.toCharArray())
    }

    fun verify(password: String, hash: String): Boolean {
        return argon2.verify(hash, password.toCharArray())
    }
}
```

---

## 5. JWT (JSON Web Tokens)

```kotlin
// ✅ CORRECTO - JJWT con validación estricta
import io.jsonwebtoken.Jwts
import io.jsonwebtoken.security.Keys
import java.util.*

@Component
class JwtTokenProvider(
    @Value("\${jwt.secret}") private val jwtSecret: String,
    @Value("\${jwt.expiration}") private val jwtExpiration: Long
) {
    private val key = Keys.hmacShaKeyFor(jwtSecret.toByteArray())

    fun generateToken(userId: String): String {
        val now = Date()
        val expiryDate = Date(now.time + jwtExpiration)

        return Jwts.builder()
            .setSubject(userId)
            .setIssuedAt(now)
            .setExpiration(expiryDate)
            .signWith(key)
            .compact()
    }

    fun validateToken(token: String): Boolean {
        return try {
            Jwts.parserBuilder()
                .setSigningKey(key)
                .build()
                .parseClaimsJws(token)
            true
        } catch (ex: Exception) {
            false
        }
    }

    fun getUserIdFromToken(token: String): String? {
        return try {
            val claims = Jwts.parserBuilder()
                .setSigningKey(key)
                .build()
                .parseClaimsJws(token)
                .body
            claims.subject
        } catch (ex: Exception) {
            null
        }
    }
}
```

---

## 6. Configuración de Seguridad (Spring Security)

```kotlin
@Configuration
@EnableWebSecurity
class SecurityConfig(
    private val jwtTokenProvider: JwtTokenProvider
) {

    @Bean
    fun filterChain(http: HttpSecurity): SecurityFilterChain {
        http {
            csrf { disable() } // Solo para APIs stateless
            sessionManagement {
                sessionCreationPolicy = SessionCreationPolicy.STATELESS
            }
            authorizeHttpRequests {
                authorize("/api/auth/**", permitAll)
                authorize("/api/admin/**", hasRole("ADMIN"))
                authorize(anyRequest, authenticated)
            }
            addFilterBefore(
                JwtAuthenticationFilter(jwtTokenProvider),
                UsernamePasswordAuthenticationFilter::class.java
            )
            headers {
                httpStrictTransportSecurity {
                    includeSubDomains = true
                    maxAgeInSeconds = 31536000
                }
                contentSecurityPolicy {
                    policyDirectives = "default-src 'self'"
                }
            }
        }
        return http.build()
    }
}
```

---

## 7. Validación de Input

```kotlin
// ✅ CORRECTO - Bean Validation
import javax.validation.constraints.*

data class UserRegistrationRequest(
    @field:NotBlank(message = "Username is required")
    @field:Size(min = 3, max = 50, message = "Username must be between 3 and 50 characters")
    @field:Pattern(regexp = "^[a-zA-Z0-9_-]+$", message = "Invalid characters in username")
    val username: String,

    @field:NotBlank(message = "Email is required")
    @field:Email(message = "Invalid email format")
    @field:Size(max = 254, message = "Email too long")
    val email: String,

    @field:NotBlank(message = "Password is required")
    @field:Size(min = 8, message = "Password must be at least 8 characters")
    @field:Pattern(
        regexp = "^(?=.*[0-9])(?=.*[a-z])(?=.*[A-Z])(?=.*[@#$%^&+=!]).*$",
        message = "Password must contain digit, lowercase, uppercase and special character"
    )
    val password: String
)

// ✅ CORRECTO - Validación manual para casos complejos
fun validateFileUpload(file: MultipartFile): Result<Unit> {
    // Validar tamaño
    if (file.size > 10 * 1024 * 1024) {
        return Result.failure(ValidationException("File too large"))
    }

    // Validar tipo MIME
    val allowedTypes = setOf("application/pdf", "image/jpeg", "image/png")
    if (file.contentType !in allowedTypes) {
        return Result.failure(ValidationException("Invalid file type"))
    }

    // Validar extensión
    val extension = file.originalFilename?.substringAfterLast('.')?.lowercase()
    val allowedExtensions = setOf("pdf", "jpg", "jpeg", "png")
    if (extension !in allowedExtensions) {
        return Result.failure(ValidationException("Invalid file extension"))
    }

    return Result.success(Unit)
}
```

---

## 8. Manejo de Excepciones

```kotlin
// ✅ CORRECTO - No exponer información interna
@RestControllerAdvice
class GlobalExceptionHandler {

    @ExceptionHandler(ValidationException::class)
    fun handleValidation(e: ValidationException): ResponseEntity<ErrorResponse> {
        return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(ErrorResponse(e.message ?: "Validation failed"))
    }

    @ExceptionHandler(Exception::class)
    fun handleGeneric(e: Exception): ResponseEntity<ErrorResponse> {
        // Log full stack trace para debugging
        logger.error("Unexpected error", e)

        // Pero no exponer al cliente
        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ErrorResponse("An unexpected error occurred"))
    }

    @ExceptionHandler(AccessDeniedException::class)
    fun handleAccessDenied(e: AccessDeniedException): ResponseEntity<ErrorResponse> {
        return ResponseEntity
            .status(HttpStatus.FORBIDDEN)
            .body(ErrorResponse("Access denied"))
    }
}
```

---

## 9. Concurrencia Segura

```kotlin
// ✅ CORRECTO - Atomic operations
import java.util.concurrent.atomic.AtomicInteger

class SafeCounter {
    private val count = AtomicInteger(0)

    fun increment(): Int = count.incrementAndGet()
    fun get(): Int = count.get()
}

// ✅ CORRECTO - Concurrent collections
import java.util.concurrent.ConcurrentHashMap

class SafeCache<K, V> {
    private val cache = ConcurrentHashMap<K, V>()

    fun put(key: K, value: V) {
        cache[key] = value
    }

    fun get(key: K): V? = cache[key]
}

// ✅ CORRECTO - Mutex para operaciones complejas
import kotlinx.coroutines.sync.Mutex
import kotlinx.coroutines.sync.withLock

class SafeAccount {
    private var balance: Double = 0.0
    private val mutex = Mutex()

    suspend fun deposit(amount: Double) {
        mutex.withLock {
            balance += amount
        }
    }

    suspend fun withdraw(amount: Double): Boolean {
        return mutex.withLock {
            if (balance >= amount) {
                balance -= amount
                true
            } else {
                false
            }
        }
    }
}
```

---

## 10. Android Específico

```kotlin
// ✅ CORRECTO - ProGuard/R8 para ofuscación
// build.gradle.kts
android {
    buildTypes {
        release {
            isMinifyEnabled = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
}

// ✅ CORRECTO - Network Security Config
// res/xml/network_security_config.xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system"/>
        </trust-anchors>
    </base-config>
</network-security-config>

// ✅ CORRECTO - Encriptación local
import androidx.security.crypto.EncryptedSharedPreferences
import androidx.security.crypto.MasterKeys

class SecureStorage(context: Context) {
    private val masterKeyAlias = MasterKeys.getOrCreate(MasterKeys.AES256_GCM_SPEC)

    private val sharedPreferences = EncryptedSharedPreferences.create(
        "secret_shared_prefs",
        masterKeyAlias,
        context,
        EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
        EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
    )

    fun saveToken(token: String) {
        sharedPreferences.edit().putString("auth_token", token).apply()
    }

    fun getToken(): String? = sharedPreferences.getString("auth_token", null)
}
```

---

## Checklist Kotlin Específico

### Null Safety
- [ ] Usar ?. (safe call) en lugar de !! (non-null assertion)
- [ ] Usar ?: (elvis operator) para defaults
- [ ] Usar let para operaciones en valores nullable
- [ ] Smart casts después de verificación null

### Seguridad
- [ ] Constructor injection en lugar de field injection
- [ ] @Valid para validación automática de input
- [ ] Queries parametrizadas (JPA/JDBC)
- [ ] BCrypt/Argon2 para passwords
- [ ] JWT con validación estricta de claims
- [ ] Spring Security con configuración explícita
- [ ] Manejo de excepciones sin exponer detalles internos
- [ ] Validación de archivos (tamaño, tipo, extensión)

### Concurrencia
- [ ] AtomicInteger/AtomicReference para contadores
- [ ] ConcurrentHashMap para caches
- [ ] Mutex para operaciones complejas
- [ ] Coroutines con manejo apropiado de errores

### Android
- [ ] ProGuard/R8 habilitado en release
- [ ] Network Security Config
- [ ] EncryptedSharedPreferences para datos sensibles
- [ ] BiometricPrompt para autenticación fuerte

---

## Referencias

- [OWASP Mobile Security](https://owasp.org/www-project-mobile-security/)
- [Spring Security Kotlin](https://docs.spring.io/spring-security/reference/)
- [Kotlin Security Guidelines](https://kotlinlang.org/docs/security.html)
