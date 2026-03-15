# Rendimiento Específico - Rust

> Guía de optimizaciones de rendimiento para código Rust. NUNCA sacrificar rendimiento obvio por "código más limpio".

---

## 1. Zero-Cost Abstractions

### El Problema

```rust
// ❌ INEFICIENTE - Abstracciones con runtime cost
fn process_items(items: &[Item]) {
    let mut result = vec![];
    for item in items {
        if item.is_active {
            result.push(item.value * 2);
        }
    }
}

// ❌ INEFICIENTE - Box innecesario
fn get_value() -> Box<i32> {
    Box::new(42) // Allocación en heap innecesaria
}
```

### La Solución

```rust
// ✅ CORRECTO - Iterators son zero-cost abstractions
fn process_items(items: &[Item]) -> Vec<i32> {
    items
        .iter()
        .filter(|item| item.is_active)
        .map(|item| item.value * 2)
        .collect()
}
// Compila a código equivalente al loop manual

// ✅ CORRECTO - Stack allocation
fn get_value() -> i32 {
    42 // Stack, sin overhead
}

// ✅ CORRECTO - Const generics para especialización
struct Array<T, const N: usize> {
    data: [T; N], // Tamaño conocido en compile-time
}

// Cada tamaño es un tipo diferente optimizado
let a: Array<i32, 3> = Array { data: [1, 2, 3] };
let b: Array<i32, 100> = Array { data: [0; 100] };

// ✅ CORRECTO - Macros para código generado
macro_rules! impl_add {
    ($($t:ty),*) => {
        $(
            impl Add for $t {
                // ... implementación
            }
        )*
    };
}

impl_add!(i8, i16, i32, i64, i128, isize);
// Genera código específico para cada tipo en compile-time
```

---

## 2. Iterator vs Loops

```rust
// ❌ INEFICIENTE - Loop manual con index
fn sum_values(items: &[i32]) -> i32 {
    let mut sum = 0;
    for i in 0..items.len() {
        sum += items[i]; // Bounds check en cada acceso
    }
    sum
}

// ✅ CORRECTO - Iterators optimizados
fn sum_values(items: &[i32]) -> i32 {
    items.iter().sum() // Sin bounds checks
}

// ✅ CORRECTO - Iterators con fold
fn process_with_state(items: &[i32]) -> i32 {
    items.iter().fold(0, |acc, &x| {
        acc + x * 2
    })
}

// ✅ CORRECTO - for_each para side effects
items.iter().for_each(|item| {
    process(item);
});

// ✅ CORRECTO - Chaining eficiente
let result: Vec<_> = items
    .par_iter()        // Parallel iterator (rayon)
    .filter(|x| x > &&0)
    .map(|x| x * 2)
    .collect();

// ✅ CORRECTO - IntoIterator para mover ownership
fn consume_items(items: Vec<Item>) {
    // into_iter() mueve los elementos, no copia
    for item in items {
        process_owned(item);
    }
}
```

---

## 3. Borrowing y Ownership

```rust
// ❌ INEFICIENTE - Clonado innecesario
fn process_data(data: String) {
    let copy = data.clone(); // Duplicación innecesaria
    // usar copy...
}

// ✅ CORRECTO - Borrowing inmutable
fn process_data(data: &str) { // String slice, no owned
    // usar data sin copiar...
}

// ✅ CORRECTO - Cow (Clone on Write)
use std::borrow::Cow;

fn format_message(input: &str) -> Cow<str> {
    if input.contains("special") {
        // Solo clonar si es necesario modificar
        Cow::Owned(input.replace("special", "normal"))
    } else {
        // Usar referencia sin copiar
        Cow::Borrowed(input)
    }
}

// ✅ CORRECTO - Slices en lugar de referencias a Vecs
// ❌ Pide Vec específico
fn process_vec(items: &Vec<i32>) {}

// ✅ Acepta cualquier slice
fn process_slice(items: &[i32]) {}

// ✅ CORRECTO - Lifetime elision
fn first_word(s: &str) -> &str { // Elided lifetimes
    s.split_whitespace().next().unwrap_or(s)
}
```

---

## 4. Arc vs Rc vs Box

```rust
// ✅ CORRECTO - Box para owned heap data
let data: Box<[u8]> = vec![1, 2, 3].into_boxed_slice();

// ✅ CORRECTO - Rc para shared ownership (single-threaded)
use std::rc::Rc;

let data: Rc<Vec<i32>> = Rc::new(vec![1, 2, 3]);
let data2 = Rc::clone(&data); // Solo incrementa refcount

// ✅ CORRECTO - Arc para shared ownership (multi-threaded)
use std::sync::Arc;
use std::thread;

let data: Arc<Vec<i32>> = Arc::new(vec![1, 2, 3]);

for _ in 0..4 {
    let data = Arc::clone(&data);
    thread::spawn(move || {
        println!("{:?}", data);
    });
}

// ✅ CORRECTO - Weak para evitar ciclos
use std::rc::{Rc, Weak};

struct Node {
    parent: Option<Weak<Node>>, // Weak evita ciclos
    children: Vec<Rc<Node>>,
}

// ✅ CORRECTO - RefCell para interior mutability
use std::cell::RefCell;

let data = RefCell::new(vec![1, 2, 3]);
{
    let mut mut_ref = data.borrow_mut();
    mut_ref.push(4);
} // Ref mut se libera aquí

let immut_ref = data.borrow();
println!("{:?}", *immut_ref);
```

---

## 5. SIMD con packed_simd

```rust
// ✅ CORRECTO - SIMD para operaciones vectoriales
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::*;

// Sumar arrays con SIMD
pub unsafe fn simd_sum(a: &[f32], b: &[f32], result: &mut [f32]) {
    // Procesar 4 f32 a la vez con AVX
    for i in (0..a.len()).step_by(4) {
        let va = _mm_loadu_ps(a.as_ptr().add(i));
        let vb = _mm_loadu_ps(b.as_ptr().add(i));
        let vr = _mm_add_ps(va, vb);
        _mm_storeu_ps(result.as_ptr().add(i) as *mut f32, vr);
    }
}

// ✅ CORRECTO - portable_simd (nightly)
#![feature(portable_simd)]
use std::simd::f32x4;

fn simd_average(values: &[f32]) -> f32 {
    let chunks = values.chunks_exact(4);
    let remainder = chunks.remainder();

    let sum = chunks.fold(f32x4::splat(0.0), |acc, chunk| {
        acc + f32x4::from_slice(chunk)
    });

    let total: f32 = sum.reduce_add() + remainder.iter().sum::<f32>();
    total / values.len() as f32
}

// ✅ CORRECTO - Auto-vectorización
// Rust puede auto-vectorizar loops simples
fn auto_vectorized_sum(values: &[f32]) -> f32 {
    values.iter().sum() // LLVM puede vectorizar esto
}
```

---

## 6. Lifetimes y Zero-Copy Parsing

```rust
// ✅ CORRECTO - Zero-copy parsing con lifetimes
#[derive(Debug)]
struct ParsedRecord<'a> {
    name: &'a str,    // Referencia al buffer original
    value: &'a str,   // Sin copias
}

fn parse_record<'a>(input: &'a str) -> Option<ParsedRecord<'a>> {
    let mut parts = input.split(',');
    Some(ParsedRecord {
        name: parts.next()?,
        value: parts.next()?,
    })
}

// ✅ CORRECTO - nom para parsing sin allocaciones
use nom::{
    IResult,
    bytes::complete::tag,
    character::complete::digit1,
    sequence::tuple,
};

fn parse_point(input: &str) -> IResult<&str, (i32, i32)> {
    let (input, (_, x, _, y, _)) = tuple((
        tag("("),
        digit1,
        tag(", "),
        digit1,
        tag(")"),
    ))(input)?;

    Ok((input, (
        x.parse().unwrap(),
        y.parse().unwrap(),
    )))
}

// ✅ CORRECTO - Memmap para archivos grandes
use memmap::MmapOptions;
use std::fs::File;

fn process_large_file(path: &str) {
    let file = File::open(path).unwrap();

    // Mapear archivo a memoria (zero-copy)
    let mmap = unsafe {
        MmapOptions::new().map(&file).unwrap()
    };

    // Procesar directamente del mmap
    for line in mmap.split(|&b| b == b'\n') {
        process_line(line);
    }
}
```

---

## 7. Async/Await con Tokio

```rust
use tokio::time::{sleep, Duration};
use tokio::task;

// ✅ CORRECTO - Async I/O eficiente
async fn fetch_all(urls: Vec<String>) -> Vec<Result<String, Error>> {
    // Crear futures para todas las requests
    let futures = urls.into_iter().map(|url| fetch(url));

    // Ejecutar todas concurrentemente
    futures::future::join_all(futures).await
}

async fn fetch(url: String) -> Result<String, Error> {
    let response = reqwest::get(&url).await?;
    response.text().await
}

// ✅ CORRECTO - Spawn para paralelismo real
async fn process_parallel(items: Vec<Item>) -> Vec<Result> {
    let mut handles = vec![];

    for item in items {
        // Cada task puede ejecutar en thread diferente
        let handle = task::spawn(async move {
            process(item).await
        });
        handles.push(handle);
    }

    let mut results = vec![];
    for handle in handles {
        results.push(handle.await.unwrap());
    }

    results
}

// ✅ CORRECTO - Channels para comunicación
use tokio::sync::mpsc;

async fn producer_consumer() {
    let (tx, mut rx) = mpsc::channel(100); // Buffer de 100

    // Producer
    tokio::spawn(async move {
        for i in 0..1000 {
            tx.send(i).await.unwrap();
        }
    });

    // Consumer
    while let Some(value) = rx.recv().await {
        process(value).await;
    }
}

// ✅ CORRECTO - Semaphore para limitar concurrencia
use tokio::sync::Semaphore;
use std::sync::Arc;

async fn limited_concurrency(urls: Vec<String>, max: usize) {
    let semaphore = Arc::new(Semaphore::new(max));

    let futures = urls.into_iter().map(|url| {
        let sem = Arc::clone(&semaphore);
        async move {
            let _permit = sem.acquire().await.unwrap();
            fetch(url).await
        }
    });

    futures::future::join_all(futures).await;
}
```

---

## 8. Profiling con cargo flamegraph

```toml
# Cargo.toml
[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }
```

```rust
// ✅ CORRECTO - Benchmarks con Criterion
use criterion::{black_box, criterion_group, criterion_main, Criterion};

fn fibonacci(n: u64) -> u64 {
    match n {
        0 => 1,
        1 => 1,
        n => fibonacci(n - 1) + fibonacci(n - 2),
    }
}

fn criterion_benchmark(c: &mut Criterion) {
    c.bench_function("fib 20", |b| {
        b.iter(|| fibonacci(black_box(20)))
    });
}

criterion_group!(benches, criterion_benchmark);
criterion_main!(benches);

// Ejecutar: cargo bench

// ✅ CORRECTO - Flamegraph
// cargo install flamegraph
// cargo flamegraph --bin myapp

// ✅ CORRECTO - Perf + Cachegrind
// cargo build --release
// perf record ./target/release/myapp
// perf report

// valgrind --tool=cachegrind ./target/release/myapp
```

---

## Checklist Rust Específico

### Ownership
- [ ] Borrowing en lugar de clonar
- [ ] Cow para Clone on Write
- [ ] Slices (&[T]) en lugar de &Vec<T>

### Smart Pointers
- [ ] Box para owned heap
- [ ] Rc para single-threaded shared
- [ ] Arc para multi-threaded shared
- [ ] RefCell para interior mutability

### Iterators
- [ ] Iterators en lugar de loops indexados
- [ ] fold, map, filter chain
- [ ] into_iter() para mover ownership

### Rendimiento
- [ ] Release builds (--release)
- [ ] Criterion para benchmarks
- [ ] Flamegraph para profiling
- [ ] #[inline] para funciones pequeñas

### Async
- [ ] Tokio para async runtime
- [ ] join_all para concurrencia
- [ ] spawn para paralelismo
- [ ] Channels para comunicación

### Zero-Copy
- [ ] Lifetimes para referencias
- [ ] nom/memmap para parsing
- [ ] evitar allocaciones innecesarias

---

## Referencias

- [Rust Performance Book](https://nnethercote.github.io/perf-book/)
- [The Rustonomicon](https://doc.rust-lang.org/nomicon/)
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/)
- [Tokio Docs](https://tokio.rs/)
