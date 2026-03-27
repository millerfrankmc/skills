# Performance Specific - Python

> Guide for performance optimizations in Python code. NEVER sacrifice obvious performance for "cleaner code".

---

## 1. GIL and Concurrency

### The Problem: Global Interpreter Lock (GIL)

```python
# ❌ INEFFICIENT - Threads for CPU-intensive (no real parallelism)
import threading

def cpu_intensive(n):
    return sum(i * i for i in range(n))

# GIL prevents real parallelism - only one thread executes at a time
threads = [
    threading.Thread(target=cpu_intensive, args=(10_000_000,))
    for _ in range(4)
]
for t in threads:
    t.start()
for t in threads:
    t.join()
```

### The Solution

```python
# ✅ CORRECT - Multiprocessing for CPU-intensive
from multiprocessing import Pool

def cpu_intensive(n):
    return sum(i * i for i in range(n))

# Real parallelism with separate processes
with Pool(processes=4) as pool:
    results = pool.map(cpu_intensive, [10_000_000] * 4)

# ✅ CORRECT - Threads only for I/O-bound
import requests
from concurrent.futures import ThreadPoolExecutor

def fetch_url(url):
    return requests.get(url).content

# Threads are efficient for I/O (release GIL during wait)
urls = ['https://api.example.com/data'] * 10
with ThreadPoolExecutor(max_workers=5) as executor:
    results = list(executor.map(fetch_url, urls))
```

---

## 2. Generators vs Lists

```python
# ❌ INEFFICIENT - Full list in memory
def process_large_file(filename):
    lines = open(filename).readlines()  # Entire list in memory
    for line in lines:
        yield line.strip().upper()

# ✅ CORRECT - Generator (constant memory)
def process_large_file(filename):
    with open(filename) as f:
        for line in f:  # One line at a time
            yield line.strip().upper()

# ✅ CORRECT - Generator expressions
# List: consumes all memory
squares_list = [x**2 for x in range(1_000_000)]

# Generator: lazy evaluation, constant memory
squares_gen = (x**2 for x in range(1_000_000))

# ✅ CORRECT - itertools for efficient operations
from itertools import islice, chain

# Process in chunks without creating full list
def chunked(iterable, n):
    iterator = iter(iterable)
    while chunk := list(islice(iterator, n)):
        yield chunk
```

---

## 3. Asyncio and Async/Await

```python
# ❌ INEFFICIENT - Sequential synchronous requests
import requests

async def fetch_all_urls(urls):
    results = []
    for url in urls:
        resp = requests.get(url)  # Blocks each time
        results.append(resp.json())
    return results

# ✅ CORRECT - aiohttp for concurrent I/O
import aiohttp
import asyncio

async def fetch_all_urls(urls):
    async with aiohttp.ClientSession() as session:
        tasks = [fetch_one(session, url) for url in urls]
        return await asyncio.gather(*tasks)

async def fetch_one(session, url):
    async with session.get(url) as resp:
        return await resp.json()

# Execution
results = asyncio.run(fetch_all_urls(urls))

# ✅ CORRECT - async for DB operations
import aiopg

async def get_users_async(user_ids):
    async with aiopg.create_pool(dsn) as pool:
        async with pool.acquire() as conn:
            async with conn.cursor() as cur:
                await cur.execute("SELECT * FROM users WHERE id = ANY(%s)", (user_ids,))
                return await cur.fetchall()
```

---

## 4. Comprehensions vs Loops

```python
# ❌ INEFFICIENT - Loop with append
result = []
for x in range(1000):
    if x % 2 == 0:
        result.append(x * 2)

# ✅ CORRECT - List comprehension (faster)
result = [x * 2 for x in range(1000) if x % 2 == 0]

# ✅ CORRECT - Dict comprehension
# Traditional loop
mapping = {}
for key, value in pairs:
    mapping[key] = value

# Dict comprehension (faster and clearer)
mapping = {key: value for key, value in pairs}

# ✅ CORRECT - setdefault vs defaultdict
from collections import defaultdict

# ❌ Slow: looks up key multiple times
for key, value in items:
    if key not in groups:
        groups[key] = []
    groups[key].append(value)

# ✅ Fast: defaultdict
groups = defaultdict(list)
for key, value in items:
    groups[key].append(value)
```

---

## 5. Caching with functools.lru_cache

```python
from functools import lru_cache

# ❌ INEFFICIENT - Recalculates fibonacci exponentially
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

# ✅ CORRECT - Automatic memoization
@lru_cache(maxsize=128)
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

# ✅ CORRECT - Cache for expensive operations
@lru_cache(maxsize=1024)
def get_user_from_db(user_id: int) -> dict:
    # Expensive: database query
    return db.query(User).get(user_id).to_dict()

# ✅ CORRECT - Cache with TTL (time-to-live)
from functools import lru_cache
import time

def ttl_cache(maxsize=128, ttl=60):
    def decorator(func):
        cache = {}
        @lru_cache(maxsize=maxsize)
        def wrapper(*args):
            key = args
            if key in cache:
                value, timestamp = cache[key]
                if time.time() - timestamp < ttl:
                    return value
            result = func(*args)
            cache[key] = (result, time.time())
            return result
        return wrapper
    return decorator
```

---

## 6. Profiling with cProfile

```python
# ✅ CORRECT - Identify bottlenecks
import cProfile
import pstats

def function_to_profile():
    # Code to analyze
    pass

# Profiling
profiler = cProfile.Profile()
profiler.enable()
function_to_profile()
profiler.disable()

# Report sorted by cumulative time
stats = pstats.Stats(profiler)
stats.sort_stats('cumulative')
stats.print_stats(20)  # Top 20 functions

# ✅ CORRECT - line_profiler for line-by-line analysis
# @profile decorator (run with kernprof -l -v script.py)
@profile
def slow_function():
    a = [i for i in range(100000)]  # Slow line
    b = [i * 2 for i in a]
    return sum(b)

# ✅ CORRECT - timeit for micro-benchmarks
import timeit

# Compare two implementations
time_list = timeit.timeit('"-".join(str(n) for n in range(100))', number=10000)
time_gen = timeit.timeit('"-".join(map(str, range(100)))', number=10000)
print(f"List comp: {time_list:.4f}s, Map: {time_gen:.4f}s")
```

---

## 7. N+1 Queries with SQLAlchemy/Django ORM

```python
# ❌ PROHIBITED - N+1 Queries
# Django
for user in User.objects.all():
    print(user.profile.bio)  # Additional query per user

# SQLAlchemy
users = session.query(User).all()
for user in users:
    print(user.orders)  # Additional query per user

# ✅ CORRECT - select_related (JOIN in one query)
# Django
users = User.objects.select_related('profile').all()
for user in users:
    print(user.profile.bio)  # No additional query

# ✅ CORRECT - prefetch_related (separate queries + cache)
# Django
users = User.objects.prefetch_related('orders').all()
for user in users:
    for order in user.orders.all():
        print(order.total)

# ✅ CORRECT - joinedload in SQLAlchemy
from sqlalchemy.orm import joinedload

users = session.query(User).options(
    joinedload(User.profile),
    joinedload(User.orders)
).all()

# ✅ CORRECT - bulk operations
# ❌ Slow: one query per object
for user in users:
    user.active = True
    user.save()

# ✅ Fast: single UPDATE query
User.objects.filter(id__in=[u.id for u in users]).update(active=True)
```

---

## 8. Efficient Data Structures

```python
# ❌ INEFFICIENT - O(n) search in list
if item in large_list:  # O(n)
    pass

# ✅ CORRECT - O(1) search in set
large_set = set(large_list)
if item in large_set:  # O(1)
    pass

# ❌ INEFFICIENT - Counting with list.count()
counts = {item: large_list.count(item) for item in set(large_list)}  # O(n²)

# ✅ CORRECT - Counter O(n)
from collections import Counter
counts = Counter(large_list)  # O(n)

# ✅ CORRECT - deque for operations at ends
from collections import deque

# ❌ List: pop(0) is O(n)
queue = [1, 2, 3]
queue.pop(0)

# ✅ Deque: popleft() is O(1)
queue = deque([1, 2, 3])
queue.popleft()

# ✅ CORRECT - namedtuple for lightweight structures
from collections import namedtuple

# Less overhead than normal class
Point = namedtuple('Point', ['x', 'y'])
p = Point(1, 2)
```

---

## 9. String Concatenation

```python
# ❌ INEFFICIENT - O(n²) due to string immutability
result = ""
for item in large_list:
    result += str(item)  # Creates new string each time

# ✅ CORRECT - join() O(n)
result = "".join(str(item) for item in large_list)

# ✅ CORRECT - io.StringIO for complex concatenation
import io

output = io.StringIO()
for item in large_list:
    output.write(str(item))
    output.write("\n")
result = output.getvalue()

# ✅ CORRECT - f-strings are faster than % or format
# Slower
"Hello %s, you have %d messages" % (name, count)
"Hello {}, you have {} messages".format(name, count)

# Faster
f"Hello {name}, you have {count} messages"
```

---

## Python Specific Checklist

### Concurrency
- [ ] Multiprocessing for CPU-bound (GIL)
- [ ] ThreadPoolExecutor for I/O-bound
- [ ] Asyncio for high I/O throughput
- [ ] NEVER use threads for intensive calculations

### Memory
- [ ] Generators for large data volumes
- [ ] `__slots__` for classes with many instances
- [ ] Weak references for large caches
- [ ] Streaming instead of loading everything in RAM

### Algorithms
- [ ] Set/Map for frequent lookups O(1)
- [ ] Deque for queues (popleft O(1))
- [ ] Counter for frequency counting
- [ ] Sort with key instead of cmp

### Database
- [ ] select_related for ForeignKey (1 query)
- [ ] prefetch_related for ManyToMany (2 queries)
- [ ] Bulk operations instead of loops
- [ ] Indexes on frequently searched columns

### Caching
- [ ] @lru_cache for pure functions
- [ ] Cache frequent query results
- [ ] Appropriate cache invalidation

### Profiling
- [ ] cProfile to identify bottlenecks
- [ ] timeit to compare implementations
- [ ] memory_profiler for memory usage

---

## References

- [Python Performance Tips](https://wiki.python.org/moin/PythonSpeed/PerformanceTips)
- [High Performance Python](https://www.oreilly.com/library/view/high-performance-python/9781449361539/)
- [Django Database Optimization](https://docs.djangoproject.com/en/stable/topics/db/optimization/)
- [SQLAlchemy Performance](https://docs.sqlalchemy.org/en/20/faq/performance.html)
