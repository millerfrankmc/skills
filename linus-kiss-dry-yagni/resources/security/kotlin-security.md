# Language-Specific Security - Kotlin

> Security protection guide for Kotlin code (JVM/Android). NEVER remove these patterns when applying KISS-DRY-YAGNI.

---

## 1. Null Safety

### The Problem
```kotlin
// ❌ Java-style null (can cause NPE)
var name: String? = null
val length = name.length // NullPointerException!

// ❌ Force non-null without checking
val name: String = nullableName!! // Crash if null
```

### The Solution
```kotlin
// ✅ CORRECT - Safe call with elvis operator
val length = name?.length ?: 0

// ✅ CORRECT - Safe call with early return
fun processUser(user: User?) {
    val name = user?.name ?: return // Early return if null
    // Process name
}

// ✅ CORRECT - Smart cast after verification
fun processUser(user: User?) {
    if (user == null) return
    // Now user is non-null (smart cast)
    println(user.name)
}

// ✅ CORRECT - let for safe scopes
user?.let { safeUser ->
    // safeUser is non-null here
    process(safeUser)
}
```

---

## 2. Secure Dependency Injection

### Spring Boot
```kotlin
// ✅ CORRECT - Constructor injection (recommended)
@Service
class UserService(
    private val userRepository: UserRepository,
    private val passwordEncoder: PasswordEncoder
) {
    fun createUser(request: CreateUserRequest): User {
        // Use injected dependencies
    }
}

// ❌ AVOID - Field injection
@Service
class BadUserService {
    @Autowired
    private lateinit var userRepository: UserRepository // May not be initialized
}

// ✅ CORRECT - Input validation
@PostMapping("/users")
fun createUser(
    @Valid @RequestBody request: CreateUserRequest
): ResponseEntity<User> {
    // Automatic validation by @Valid
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
// ❌ FORBIDDEN - String concatenation
@Query("SELECT * FROM users WHERE name = '$name'")
fun findByName(name: String): List<User>

// ✅ CORRECT - Named parameters
@Query("SELECT u FROM User u WHERE u.name = :name")
fun findByName(@Param("name") name: String): List<User>

// ✅ CORRECT - Derived methods (Spring generates safe query)
fun findByUsernameAndActive(username: String, active: Boolean): List<User>

// ✅ CORRECT - Criteria API for dynamic queries
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

// ✅ CORRECT - JPA Repository with safe JPQL
interface UserRepository : JpaRepository<User, Long> {
    @Query("SELECT u FROM User u WHERE u.email = ?1")
    fun findByEmail(email: String): User?
}
```

---

## 4. Password Hashing

```kotlin
// ✅ CORRECT - BCrypt (Spring Security)
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

// ✅ CORRECT - Argon2 (more modern)
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
// ✅ CORRECT - JJWT with strict validation
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

## 6. Security Configuration (Spring Security)

```kotlin
@Configuration
@EnableWebSecurity
class SecurityConfig(
    private val jwtTokenProvider: JwtTokenProvider
) {

    @Bean
    fun filterChain(http: HttpSecurity): SecurityFilterChain {
        http {
            csrf { disable() } // Only for stateless APIs
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

## 7. Input Validation

```kotlin
// ✅ CORRECT - Bean Validation
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

// ✅ CORRECT - Manual validation for complex cases
fun validateFileUpload(file: MultipartFile): Result<Unit> {
    // Validate size
    if (file.size > 10 * 1024 * 1024) {
        return Result.failure(ValidationException("File too large"))
    }

    // Validate MIME type
    val allowedTypes = setOf("application/pdf", "image/jpeg", "image/png")
    if (file.contentType !in allowedTypes) {
        return Result.failure(ValidationException("Invalid file type"))
    }

    // Validate extension
    val extension = file.originalFilename?.substringAfterLast('.')?.lowercase()
    val allowedExtensions = setOf("pdf", "jpg", "jpeg", "png")
    if (extension !in allowedExtensions) {
        return Result.failure(ValidationException("Invalid file extension"))
    }

    return Result.success(Unit)
}
```

---

## 8. Exception Handling

```kotlin
// ✅ CORRECT - Don't expose internal information
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
        // Log full stack trace for debugging
        logger.error("Unexpected error", e)

        // But don't expose to client
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

## 9. Safe Concurrency

```kotlin
// ✅ CORRECT - Atomic operations
import java.util.concurrent.atomic.AtomicInteger

class SafeCounter {
    private val count = AtomicInteger(0)

    fun increment(): Int = count.incrementAndGet()
    fun get(): Int = count.get()
}

// ✅ CORRECT - Concurrent collections
import java.util.concurrent.ConcurrentHashMap

class SafeCache<K, V> {
    private val cache = ConcurrentHashMap<K, V>()

    fun put(key: K, value: V) {
        cache[key] = value
    }

    fun get(key: K): V? = cache[key]
}

// ✅ CORRECT - Mutex for complex operations
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

## 10. Android Specific

```kotlin
// ✅ CORRECT - ProGuard/R8 for obfuscation
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

// ✅ CORRECT - Network Security Config
// res/xml/network_security_config.xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system"/>
        </trust-anchors>
    </base-config>
</network-security-config>

// ✅ CORRECT - Local encryption
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

## Kotlin-Specific Checklist

### Null Safety
- [ ] Use ?. (safe call) instead of !! (non-null assertion)
- [ ] Use ?: (elvis operator) for defaults
- [ ] Use let for operations on nullable values
- [ ] Smart casts after null check

### Security
- [ ] Constructor injection instead of field injection
- [ ] @Valid for automatic input validation
- [ ] Parameterized queries (JPA/JDBC)
- [ ] BCrypt/Argon2 for passwords
- [ ] JWT with strict claims validation
- [ ] Spring Security with explicit configuration
- [ ] Exception handling without exposing internal details
- [ ] File validation (size, type, extension)

### Concurrency
- [ ] AtomicInteger/AtomicReference for counters
- [ ] ConcurrentHashMap for caches
- [ ] Mutex for complex operations
- [ ] Coroutines with proper error handling

### Android
- [ ] ProGuard/R8 enabled in release
- [ ] Network Security Config
- [ ] EncryptedSharedPreferences for sensitive data
- [ ] BiometricPrompt for strong authentication

---

## References

- [OWASP Mobile Security](https://owasp.org/www-project-mobile-security/)
- [Spring Security Kotlin](https://docs.spring.io/spring-security/reference/)
- [Kotlin Security Guidelines](https://kotlinlang.org/docs/security.html)
