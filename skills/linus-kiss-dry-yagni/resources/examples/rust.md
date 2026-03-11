# Ejemplos Rust

## Anti-Clever Code

**MAL**:
```rust
fn process(items: Vec<Option<&str>>) -> Vec<String> {
    items.iter().filter_map(|x| x.map(|s| s.to_uppercase())).filter(|s| !s.is_empty()).collect()
}
```

**BIEN**:
```rust
fn process_items(items: Vec<Option<&str>>) -> Vec<String> {
    items
        .iter()
        .filter_map(|opt| opt.map(|s| s.to_uppercase()))
        .filter(|s| !s.is_empty())
        .collect()
}
```

---

## YAGNI - Eliminar Abstracción Innecesaria

**MAL**:
```rust
trait Processor<T> {
    fn process(&self, data: T) -> T;
}

struct StringProcessor;

impl Processor<String> for StringProcessor {
    fn process(&self, data: String) -> String {
        data.to_uppercase()
    }
}
```

**BIEN**:
```rust
fn process_string(data: String) -> String {
    data.to_uppercase()
}
```

---

## KISS - Guard Clauses

**MAL** (anidado):
```rust
fn process_order(order: Option<&Order>) -> Result<(), Error> {
    if let Some(order) = order {
        if order.is_valid() {
            if order.has_items() {
                if let Some(payment) = &order.payment {
                    if payment.is_processed {
                        return fulfill_order(order);
                    }
                }
            }
        }
    }
    Err(Error::InvalidOrder)
}
```

**BIEN** (guard clauses):
```rust
fn process_order(order: Option<&Order>) -> Result<(), Error> {
    let order = order.ok_or(Error::InvalidOrder)?;
    if !order.is_valid() { return Err(Error::InvalidOrder); }
    if !order.has_items() { return Err(Error::NoItems); }
    let payment = order.payment.as_ref().ok_or(Error::NoPayment)?;
    if !payment.is_processed { return Err(Error::PaymentNotProcessed); }

    fulfill_order(order)
}
```

---

## DRY - Centralizar

**MAL** (duplicado):
```rust
fn validate_email(email: &str) -> bool {
    email.contains('@') && email.contains('.')
}

fn validate_user_email(user: &User) -> bool {
    user.email.contains('@') && user.email.contains('.')
}
```

**BIEN**:
```rust
fn is_valid_email(email: &str) -> bool {
    email.contains('@') && email.contains('.')
}

fn validate_user_email(user: &User) -> bool {
    is_valid_email(&user.email)
}
```

---

## Error Handling Simple

**MAL** (over-engineered):
```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum UserError {
    #[error("validation failed: {0}")]
    ValidationError(String),
    #[error("not found")]
    NotFound,
    #[error("database error")]
    DatabaseError,
}

pub type Result<T> = std::result::Result<T, UserError>;
```

**BIEN**:
```rust
#[derive(Debug)]
pub enum Error {
    InvalidInput,
    NotFound,
}

pub type Result<T> = std::result::Result<T, Error>;
```

---

## Structs vs Tuplas Nombradas

**MAL** (parámetros sueltos):
```rust
fn create_user(name: &str, email: &str, age: u32, active: bool) -> User {
    // ¿Qué orden? ¿Fácil de confundir?
}
```

**BIEN** (struct):
```rust
struct UserData<'a> {
    name: &'a str,
    email: &'a str,
    age: u32,
    active: bool,
}

fn create_user(data: UserData) -> User {
    // Claro y auto-documentado
}
```

---

## Match vs if let

**MAL** (match innecesario):
```rust
match maybe_user {
    Some(user) => {
        match user.email {
            Some(email) => println!("{}", email),
            None => {}
        }
    }
    None => {}
}
```

**BIEN** (if let):
```rust
if let Some(user) = maybe_user {
    if let Some(email) = &user.email {
        println!("{}", email);
    }
}
```

---

## Option/Result - Evitar unwrap

**MAL**:
```rust
let user = find_user(id).unwrap();
let email = user.email.unwrap();
```

**BIEN**:
```rust
let email = find_user(id)
    .and_then(|u| u.email)
    .ok_or("User or email not found")?;
```
