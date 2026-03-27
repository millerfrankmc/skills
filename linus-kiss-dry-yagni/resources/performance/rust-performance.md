# Performance Specific - Rust

> Guide for performance optimizations in Rust code. NEVER sacrifice obvious performance for "cleaner code".

---

## 1. Zero-Cost Abstractions

### The Problem

```rust
// ❌ INEFFICIENT - Abstractions with runtime cost
fn process_items(items: &[Item]) {
    let mut result = vec![];
    for item in items {
        if item.is_active {
            result.push(item.value * 2);
        }
    }
}

// ❌ INEFFICIENT - Unnecessary Box
fn get_value() -> Box<i32> {
    Box::new(42) // Unnecessary heap allocation
}
```

### The Solution

```rust
// ✅ CORRECT - Iterators are zero-cost abstractions
fn process_items(items: &[Item]) -> Vec<i32> {
    items
        .iter()
        .filter(|item| item.is_active)
        .map(|item| item.value * 2)
        .collect()
}
// Compiles to code equivalent to manual loop

// ✅ CORRECT - Stack allocation
fn get_value() -> i32 {
    42 // Stack, no overhead
}

// ✅ CORRECT - Const generics for specialization
struct Array<T, const N: usize> {
    data: [T; N], // Size known at compile-time
}

// Each size is a different optimized type
let a: Array<i32, 3> = Array { data: [1, 2, 3] };
let b: Array<i32, 100> = Array { data: [0; 100] };

// ✅ CORRECT - Macros for generated code
macro_rules! impl_add {
    ($($t:ty),*) => {
        $(
            impl Add for $t {
                // ... implementation
            }
        )*
    };
}

impl_add!(i8, i16, i32, i64, i128, isize);
// Generates specific code for each type at compile-time
```

---

## 2. Iterator vs Loops

```rust
// ❌ INEFFICIENT - Manual loop with index
fn sum_values(items: &[i32]) -> i32 {
    let mut sum = 0;
    for i in 0..items.len() {
        sum += items[i]; // Bounds check on each access
    }
    sum
}

// ✅ CORRECT - Optimized iterators
fn sum_values(items: &[i32]) -> i32 {
    items.iter().sum() // No bounds checks
}

// ✅ CORRECT - Iterators with fold
fn process_with_state(items: &[i32]) -> i32 {
    items.iter().fold(0, |acc, &x| {
        acc + x * 2
    })
}

// ✅ CORRECT - for_each for side effects
items.iter().for_each(|item| {
    process(item);
});

// ✅ CORRECT - Efficient chaining
let result: Vec<_> = items
    .par_iter()        // Parallel iterator (rayon)
    .filter(|x| x > &&0)
    .map(|x| x * 2)
    .collect();

// ✅ CORRECT - IntoIterator for moving ownership
fn consume_items(items: Vec<Item>) {
    // into_iter() moves elements, doesn't copy
    for item in items {
        process_owned(item);
    }
}
```

---

## 3. Borrowing and Ownership

```rust
// ❌ INEFFICIENT - Unnecessary cloning
fn process_data(data: String) {
    let copy = data.clone(); // Unnecessary duplication
    // use copy...
}

// ✅ CORRECT - Immutable borrowing
fn process_data(data: &str) { // String slice, not owned
    // use data without copying...
}

// ✅ CORRECT - Cow (Clone on Write)
use std::borrow::Cow;

fn format_message(input: &str) -> Cow<str> {
    if input.contains("special") {
        // Only clone if modification needed
        Cow::Owned(input.replace("special", "normal"))
    } else {
        // Use reference without copying
        Cow::Borrowed(input)
    }
}

// ✅ CORRECT - Slices instead of Vec references
// ❌ Asks for specific Vec
fn process_vec(items: &Vec<i32>) {}

// ✅ Accepts any slice
fn process_slice(items: &[i32]) {}

// ✅ CORRECT - Lifetime elision
fn first_word(s: &str) -> &str { // Elided lifetimes
    s.split_whitespace().next().unwrap_or(s)
}
```

---

## 4. Arc vs Rc vs Box

```rust
// ✅ CORRECT - Box for owned heap data
let data: Box<[u8]> = vec![1, 2, 3].into_boxed_slice();

// ✅ CORRECT - Rc for shared ownership (single-threaded)
use std::rc::Rc;

let data: Rc<Vec<i32>> = Rc::new(vec![1, 2, 3]);
let data2 = Rc::clone(&data); // Just increments refcount

// ✅ CORRECT - Arc for shared ownership (multi-threaded)
use std::sync::Arc;
use std::thread;

let data: Arc<Vec<i32>> = Arc::new(vec![1, 2, 3]);

for _ in 0..4 {
    let data = Arc::clone(&data);
    thread::spawn(move || {
        println!("{:?}", data);
    });
}

// ✅ CORRECT - Weak to avoid cycles
use std::rc::{Rc, Weak};

struct Node {
    parent: Option<Weak<Node>>, // Weak avoids cycles
    children: Vec<Rc<Node>>,
}

// ✅ CORRECT - RefCell for interior mutability
use std::cell::RefCell;

let data = RefCell::new(vec![1, 2, 3]);
{
    let mut mut_ref = data.borrow_mut();
    mut_ref.push(4);
} // Ref mut released here

let immut_ref = data.borrow();
println!("{:?}", *immut_ref);
```

---

## 5. SIMD with packed_simd

```rust
// ✅ CORRECT - SIMD for vector operations
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::*;

// Sum arrays with SIMD
pub unsafe fn simd_sum(a: &[f32], b: &[f32], result: &mut [f32]) {
    // Process 4 f32 at once with AVX
    for i in (0..a.len()).step_by(4) {
        let va = _mm_loadu_ps(a.as_ptr().add(i));
        let vb = _mm_loadu_ps(b.as_ptr().add(i));
        let vr = _mm_add_ps(va, vb);
        _mm_storeu_ps(result.as_ptr().add(i) as *mut f32, vr);
    }
}

// ✅ CORRECT - portable_simd (nightly)
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

// ✅ CORRECT - Auto-vectorization
// Rust can auto-vectorize simple loops
fn auto_vectorized_sum(values: &[f32]) -> f32 {
    values.iter().sum() // LLVM can vectorize this
}
```

---

## 6. Lifetimes and Zero-Copy Parsing

```rust
// ✅ CORRECT - Zero-copy parsing with lifetimes
#[derive(Debug)]
struct ParsedRecord<'a> {
    name: &'a str,    // Reference to original buffer
    value: &'a str,   // No copies
}

fn parse_record<'a>(input: &'a str) -> Option<ParsedRecord<'a>> {
    let mut parts = input.split(',');
    Some(ParsedRecord {
        name: parts.next()?,
        value: parts.next()?,
    })
}

// ✅ CORRECT - nom for parsing without allocations
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

// ✅ CORRECT - Memmap for large files
use memmap::MmapOptions;
use std::fs::File;

fn process_large_file(path: &str) {
    let file = File::open(path).unwrap();

    // Map file to memory (zero-copy)
    let mmap = unsafe {
        MmapOptions::new().map(&file).unwrap()
    };

    // Process directly from mmap
    for line in mmap.split(|&b| b == b'\n') {
        process_line(line);
    }
}
```

---

## 7. Async/Await with Tokio

```rust
use tokio::time::{sleep, Duration};
use tokio::task;

// ✅ CORRECT - Efficient async I/O
async fn fetch_all(urls: Vec<String>) -> Vec<Result<String, Error>> {
    // Create futures for all requests
    let futures = urls.into_iter().map(|url| fetch(url));

    // Execute all concurrently
    futures::future::join_all(futures).await
}

async fn fetch(url: String) -> Result<String, Error> {
    let response = reqwest::get(&url).await?;
    response.text().await
}

// ✅ CORRECT - Spawn for real parallelism
async fn process_parallel(items: Vec<Item>) -> Vec<Result> {
    let mut handles = vec![];

    for item in items {
        // Each task can run on different thread
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

// ✅ CORRECT - Channels for communication
use tokio::sync::mpsc;

async fn producer_consumer() {
    let (tx, mut rx) = mpsc::channel(100); // Buffer of 100

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

// ✅ CORRECT - Semaphore to limit concurrency
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

## 8. Profiling with cargo flamegraph

```toml
# Cargo.toml
[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }
```

```rust
// ✅ CORRECT - Benchmarks with Criterion
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

// Run: cargo bench

// ✅ CORRECT - Flamegraph
// cargo install flamegraph
// cargo flamegraph --bin myapp

// ✅ CORRECT - Perf + Cachegrind
// cargo build --release
// perf record ./target/release/myapp
// perf report

// valgrind --tool=cachegrind ./target/release/myapp
```

---

## Rust Specific Checklist

### Ownership
- [ ] Borrowing instead of cloning
- [ ] Cow for Clone on Write
- [ ] Slices (&[T]) instead of &Vec<T>

### Smart Pointers
- [ ] Box for owned heap
- [ ] Rc for single-threaded shared
- [ ] Arc for multi-threaded shared
- [ ] RefCell for interior mutability

### Iterators
- [ ] Iterators instead of index loops
- [ ] fold, map, filter chain
- [ ] into_iter() for moving ownership

### Performance
- [ ] Release builds (--release)
- [ ] Criterion for benchmarks
- [ ] Flamegraph for profiling
- [ ] #[inline] for small functions

### Async
- [ ] Tokio for async runtime
- [ ] join_all for concurrency
- [ ] spawn for parallelism
- [ ] Channels for communication

### Zero-Copy
- [ ] Lifetimes for references
- [ ] nom/memmap for parsing
- [ ] Avoid unnecessary allocations

---

## References

- [Rust Performance Book](https://nnethercote.github.io/perf-book/)
- [The Rustonomicon](https://doc.rust-lang.org/nomicon/)
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/)
- [Tokio Docs](https://tokio.rs/)
