# Language-Specific Security - Rust

> Security protection guide for Rust code. NEVER remove these patterns when applying KISS-DRY-YAGNI.
>
> **Special note**: Rust has memory safety by design, but there are `unsafe` patterns and common errors that can compromise security.

---

## 1. Unsafe Code

### The Problem
```rust
// ❌ FORBIDDEN - Unnecessary use of unsafe
unsafe fn dangerous() {
    let ptr = 0x12345 as *mut i32;
    *ptr = 42; // Potential segmentation fault
}

// ❌ FORBIDDEN - Raw pointers without validation
unsafe fn process_data(ptr: *const u8, len: usize) {
    let slice = std::slice::from_raw_parts(ptr, len);
    // If ptr is null or len is incorrect → UB
}
```

### The Solution
```rust
// ✅ CORRECT - Avoid unsafe when possible
fn safe_process(data: &[u8]) -> Result<(), Error> {
    // The borrow checker guarantees validity
    process(data)
}

// ✅ CORRECT - If inevitable, encapsulate and validate
/// # Safety
/// `ptr` must be valid and non-null
/// `len` must be the correct buffer size
pub unsafe fn process_unchecked(ptr: *const u8, len: usize) {
    // Document safety contracts
    assert!(!ptr.is_null(), "ptr must not be null");
    let slice = std::slice::from_raw_parts(ptr, len);
    process(slice);
}

// ✅ CORRECT - Safe wrapper around unsafe
pub fn process_safe(data: &[u8]) -> Result<(), Error> {
    if data.is_empty() {
        return Err(Error::EmptyInput);
    }

    // unsafe encapsulated, not exposed to user
    unsafe {
        process_unchecked(data.as_ptr(), data.len());
    }
    Ok(())
}

// ✅ CORRECT - Use safe abstractions
use std::sync::Mutex; // Instead of manual spinlocks

// Instead of pthreads/raw threads
use std::thread;

// Instead of malloc/free
use Box::new(val);
```

### When Unsafe is Acceptable
```rust
// ✅ FFI bindings
extern "C" {
    fn c_library_function(ptr: *const c_char);
}

pub fn safe_wrapper(input: &str) {
    let c_string = CString::new(input).expect("valid C string");
    unsafe {
        c_library_function(c_string.as_ptr());
    }
}

// ✅ Zero-copy parsing with validation
use bytes::Bytes;

fn parse_header(data: Bytes) -> Result<Header, Error> {
    if data.len() < HEADER_SIZE {
        return Err(Error::TooShort);
    }

    // Safe because we validated size
    let header = unsafe {
        &*(data.as_ptr() as *const Header)
    };

    Ok(*header)
}
```

---

## 2. FFI (Foreign Function Interface)

```rust
// ❌ FORBIDDEN - C strings without validation
pub extern "C" fn process_string(ptr: *const c_char) {
    let c_str = unsafe { CStr::from_ptr(ptr) }; // Crash if ptr is null
    // ...
}

// ✅ CORRECT - Validate before using
use std::ffi::{CStr, CString};
use std::os::raw::c_char;

#[no_mangle]
pub extern "C" fn process_string_safe(ptr: *const c_char) -> i32 {
    if ptr.is_null() {
        return -1; // Error code
    }

    let c_str = unsafe { CStr::from_ptr(ptr) };
    match c_str.to_str() {
        Ok(s) => {
            process(s);
            0 // Success
        }
        Err(_) => -2, // Invalid UTF-8
    }
}

// ✅ CORRECT - Panic safety in FFI
use std::panic::catch_unwind;

#[no_mangle]
pub extern "C" fn safe_entry_point() -> i32 {
    match catch_unwind(|| {
        // Rust code that might panic
        risky_operation()
    }) {
        Ok(result) => result,
        Err(_) => {
            // Panic occurred, return error
            -1
        }
    }
}
```

---

## 3. Panic Safety

```rust
// ❌ FORBIDDEN - Drop that can panic
impl Drop for Resource {
    fn drop(&mut self) {
        self.cleanup().expect("cleanup failed"); // Panic in drop!
    }
}

// ✅ CORRECT - Drop that never panics
impl Drop for Resource {
    fn drop(&mut self) {
        if let Err(e) = self.cleanup() {
            // Log error but don't panic
            eprintln!("Cleanup failed: {}", e);
        }
    }
}

// ✅ CORRECT - std::mem::forget for special cases
use std::mem;

fn transfer_ownership(val: Box<Resource>) {
    let raw = Box::into_raw(val);

    // If it fails, we don't want drop to execute
    if unsafe { transfer_to_c(raw) } {
        mem::forget(val); // Successfully transferred
    } else {
        // Reconstruct Box so drop happens
        unsafe { Box::from_raw(raw) };
    }
}
```

---

## 4. Secrets Management

```rust
// ❌ FORBIDDEN - Strings for secrets (can remain in memory)
let password = String::from("secret123");

// ✅ CORRECT - zeroize to clear memory
use zeroize::{Zeroize, ZeroizeOnDrop};

#[derive(Zeroize, ZeroizeOnDrop)]
struct Secret {
    #[zeroize(skip)] // Don't clear the ID
    id: String,
    value: Vec<u8>, // This is cleared with zeros
}

// ✅ CORRECT - Mlock to prevent swap to disk
use secrets::SecretBox;

fn store_secret(data: &[u8]) -> SecretBox<[u8]> {
    let mut secret = SecretBox::new(|s| {
        s.copy_from_slice(data);
    });
    secret
}

// ✅ CORRECT - Environment variables
use std::env;

fn get_api_key() -> Result<String, VarError> {
    env::var("API_KEY")
}

// ✅ CORRECT - dotenv for development
use dotenv::dotenv;

fn init() {
    dotenv().ok(); // Load .env file
    let key = env::var("API_KEY").expect("API_KEY must be set");
}
```

---

## 5. Safe Deserialization

```rust
// ❌ FORBIDDEN - Deserialization without limits
#[derive(Deserialize)]
struct Config {
    data: Vec<u8>, // Can be any size!
}

// ✅ CORRECT - Size validation with serde
use serde::Deserialize;

#[derive(Deserialize)]
struct Config {
    #[serde(deserialize_with = "deserialize_bounded")]
    data: Vec<u8>,
}

fn deserialize_bounded<'de, D>(deserializer: D) -> Result<Vec<u8>, D::Error>
where
    D: serde::Deserializer<'de>,
{
    let vec = Vec::deserialize(deserializer)?;
    if vec.len() > MAX_SIZE {
        return Err(serde::de::Error::custom("data too large"));
    }
    Ok(vec)
}

// ✅ CORRECT - Safe JSON deserialization
use serde_json;

fn parse_json_safe(input: &str) -> Result<Value, Error> {
    // Limit nesting depth
    let deserializer = serde_json::Deserializer::from_str(input);
    let value = serde_json::Value::deserialize(deserializer)?;

    // Validate size after parsing
    if serde_json::to_string(&value)?.len() > MAX_JSON_SIZE {
        return Err(Error::TooLarge);
    }

    Ok(value)
}
```

---

## 6. SQL Injection

```rust
// ❌ FORBIDDEN - String formatting
let query = format!("SELECT * FROM users WHERE id = {}", user_id);

// ✅ CORRECT - sqlx with parameterized queries
use sqlx::query_as;

let user: User = query_as(
    "SELECT id, name, email FROM users WHERE id = ?"
)
.bind(user_id)
.fetch_one(&pool)
.await?;

// ✅ CORRECT - sea-orm (type-safe ORM)
use sea_orm::{entity::*, query::*};

let user = User::find_by_id(user_id)
    .one(&db)
    .await?;

// ✅ CORRECT - Diesel (compile-time checked ORM)
use diesel::prelude::*;

let results = users
    .filter(id.eq(user_id))
    .load::<User>(&connection)?;

// ✅ CORRECT - tokio-postgres
let row = client
    .query_one(
        "SELECT name FROM users WHERE id = $1",
        &[&user_id]
    )
    .await?;
```

---

## 7. Secure HTTP (Axum/Actix/Rocket)

### Axum
```rust
use axum::{
    routing::post,
    Router,
    extract::State,
    http::StatusCode,
};
use tower_http::{
    limit::RequestBodyLimitLayer,
    timeout::TimeoutLayer,
};

// ✅ CORRECT - Timeouts and limits
let app = Router::new()
    .route("/upload", post(upload_handler))
    .layer(RequestBodyLimitLayer::new(10 * 1024 * 1024)) // 10MB max
    .layer(TimeoutLayer::new(Duration::from_secs(30)))
    .with_state(pool);

// ✅ CORRECT - Input validation
use validator::Validate;

#[derive(Deserialize, Validate)]
struct CreateUser {
    #[validate(length(min = 3, max = 50))]
    username: String,

    #[validate(email)]
    email: String,

    #[validate(length(min = 8))]
    password: String,
}

async fn create_user(
    Json(payload): Json<CreateUser>,
) -> Result<StatusCode, (StatusCode, String)> {
    payload.validate()
        .map_err(|e| (StatusCode::BAD_REQUEST, e.to_string()))?;

    // Process...
    Ok(StatusCode::CREATED)
}
```

### Actix
```rust
use actix_web::{web, App, HttpServer, HttpResponse};
use actix_web::middleware::{Logger, Compress, DefaultHeaders};

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .wrap(Logger::default())
            .wrap(Compress::default())
            .wrap(DefaultHeaders::new().add(("X-Frame-Options", "DENY")))
            .service(
                web::resource("/api/{name}")
                    .route(web::get().to(greet))
                    .route(web::post().to(create))
            )
    })
    .bind("127.0.0.1:8080")?
    .workers(4)
    .shutdown_timeout(60) // Graceful shutdown
    .run()
    .await
}
```

---

## 8. Cryptography

### Password Hashing
```rust
// ✅ CORRECT - argon2 (recommended)
use argon2::{
    password_hash::{
        rand_core::OsRng,
        PasswordHash, PasswordHasher, PasswordVerifier, SaltString
    },
    Argon2,
};

fn hash_password(password: &str) -> Result<String, argon2::Error> {
    let salt = SaltString::generate(&mut OsRng);
    let argon2 = Argon2::default();

    let password_hash = argon2
        .hash_password(password.as_bytes(), &salt)?
        .to_string();

    Ok(password_hash)
}

fn verify_password(password: &str, hash: &str) -> Result<bool, argon2::Error> {
    let parsed_hash = PasswordHash::new(hash)?;
    let argon2 = Argon2::default();

    Ok(argon2.verify_password(password.as_bytes(), &parsed_hash).is_ok())
}

// ✅ CORRECT - ring for general cryptography
use ring::{
    aead::{Aes256Gcm, Nonce, UnboundKey, AES_256_GCM},
    rand::{SecureRandom, SystemRandom},
};

fn encrypt_data(plaintext: &[u8], key: &[u8]) -> Result<Vec<u8>, Error> {
    let unbound_key = UnboundKey::new(&AES_256_GCM, key)?;
    let key = aes_gcm::Key::from_slice(key);
    let cipher = Aes256Gcm::new(key);

    let nonce = {
        let rng = SystemRandom::new();
        let mut nonce = [0u8; 12];
        rng.fill(&mut nonce)?;
        Nonce::assume_unique_for_key(nonce)
    };

    // ... encryption
    Ok(ciphertext)
}
```

### JWT
```rust
use jsonwebtoken::{decode, encode, Header, Validation, Algorithm};
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct Claims {
    sub: String,        // Subject (user id)
    exp: usize,         // Expiration time
    iat: usize,         // Issued at
}

fn create_token(user_id: &str, secret: &[u8]) -> Result<String, Error> {
    let now = chrono::Utc::now().timestamp() as usize;
    let claims = Claims {
        sub: user_id.to_owned(),
        exp: now + 86400, // 24 hours
        iat: now,
    };

    encode(&Header::default(), &claims, &EncodingKey::from_secret(secret))
        .map_err(|e| Error::TokenCreation(e.to_string()))
}

fn verify_token(token: &str, secret: &[u8]) -> Result<Claims, Error> {
    let mut validation = Validation::new(Algorithm::HS256);
    validation.validate_exp = true;
    validation.validate_iat = true;
    validation.required_spec_claims = ["exp", "iat", "sub"].iter().cloned().collect();

    decode::<Claims>(token, &DecodingKey::from_secret(secret), &validation)
        .map(|data| data.claims)
        .map_err(|e| Error::InvalidToken(e.to_string()))
}
```

---

## 9. Safe Concurrency

```rust
// ✅ CORRECT - Arc + Mutex for shared data
use std::sync::{Arc, Mutex};

let data = Arc::new(Mutex::new(0));
let mut handles = vec![];

for _ in 0..10 {
    let data = Arc::clone(&data);
    let handle = thread::spawn(move || {
        let mut num = data.lock().unwrap();
        *num += 1;
    });
    handles.push(handle);
}

// ✅ CORRECT - RwLock for frequent reads
use std::sync::RwLock;

let config = Arc::new(RwLock::new(Config::default()));

// Many readers
let read_config = Arc::clone(&config);
thread::spawn(move || {
    let cfg = read_config.read().unwrap();
    println!("{:?}", *cfg);
});

// Few writers
let write_config = Arc::clone(&config);
thread::spawn(move || {
    let mut cfg = write_config.write().unwrap();
    cfg.update();
});

// ✅ CORRECT - tokio::sync for async
use tokio::sync::{RwLock, Mutex};

async fn process_data(data: Arc<RwLock<Data>>) {
    // Async lock, doesn't block thread
    let guard = data.read().await;
    process(&*guard).await;
}

// ✅ CORRECT - Channels for communication
use tokio::sync::mpsc;

let (tx, mut rx) = mpsc::channel(100);

tokio::spawn(async move {
    while let Some(msg) = rx.recv().await {
        process(msg).await;
    }
});
```

---

## 10. Input Validation

```rust
use validator::{Validate, ValidationError};

#[derive(Debug, Validate)]
struct UserRegistration {
    #[validate(length(min = 3, max = 50), custom = "validate_username")]
    username: String,

    #[validate(email)]
    email: String,

    #[validate(length(min = 8), custom = "validate_password_strength")]
    password: String,

    #[validate(range(min = 0, max = 150))]
    age: u8,
}

fn validate_username(username: &str) -> Result<(), ValidationError> {
    if !username.chars().all(|c| c.is_alphanumeric() || c == '_' || c == '-') {
        return Err(ValidationError::new("invalid_chars"));
    }
    Ok(())
}

fn validate_password_strength(password: &str) -> Result<(), ValidationError> {
    let has_upper = password.chars().any(|c| c.is_uppercase());
    let has_lower = password.chars().any(|c| c.is_lowercase());
    let has_digit = password.chars().any(|c| c.is_ascii_digit());
    let has_special = password.chars().any(|c| !c.is_alphanumeric());

    if !has_upper || !has_lower || !has_digit || !has_special {
        return Err(ValidationError::new("not_strong_enough"));
    }
    Ok(())
}

// ✅ CORRECT - File validation
use mime::Mime;

fn validate_file(data: &[u8], declared_mime: &str) -> Result<(), Error> {
    // Validate size
    if data.len() > MAX_FILE_SIZE {
        return Err(Error::FileTooLarge);
    }

    // Validate magic bytes
    let detected = infer::get(data)
        .ok_or(Error::UnknownFileType)?;

    if detected.mime_type() != declared_mime {
        return Err(Error::MimeMismatch);
    }

    Ok(())
}
```

---

## Rust-Specific Checklist

### Unsafe Code
- [ ] Avoid unsafe when possible
- [ ] Document safety contracts in comments
- [ ] Validate preconditions (null checks, bounds)
- [ ] Encapsulate unsafe in safe APIs
- [ ] Use safe abstractions (Mutex vs spinlocks)

### Security
- [ ] Environment variables for secrets (no hardcoding)
- [ ] zeroize to clear secrets from memory
- [ ] Parameterized queries (sqlx, diesel, sea-orm)
- [ ] Input validation with validator
- [ ] Passwords with argon2
- [ ] JWT with strict validation
- [ ] Timeouts on I/O operations
- [ ] Size limits on deserialization

### FFI
- [ ] Validate null pointers before using
- [ ] panic::catch_unwind at FFI boundaries
- [ ] CString::new() for converting to C strings
- [ ] Document safety contracts

### Concurrency
- [ ] Arc for shared ownership between threads
- [ ] Mutex/RwLock for shared mutable data
- [ ] tokio::sync for async
- [ ] Channels for inter-thread communication
- [ ] Avoid deadlocks (consistent lock ordering)

### Resources
- [ ] Graceful shutdown with tokio::signal
- [ ] Request body limits (tower-http)
- [ ] Timeouts in HTTP clients
- [ ] Drop impls that never panic

---

## References

- [Rust Security Guidelines](https://anixe.github.io/rust-security-guidelines/)
- [Rust Secure Coding](https://github.com/rust-secure-code/safety-dance)
- [Rustonomicon - Unsafe](https://doc.rust-lang.org/nomicon/)
- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/)
