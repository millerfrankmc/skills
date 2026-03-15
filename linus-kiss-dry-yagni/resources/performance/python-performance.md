# Rendimiento Específico - Python

> Guía de optimizaciones de rendimiento para código Python. NUNCA sacrificar rendimiento obvio por "código más limpio".

---

## 1. GIL y Concurrencia

### El Problema: Global Interpreter Lock (GIL)

```python
# ❌ INEFICIENTE - Threads para CPU-intensive (no paralelismo real)
import threading

def cpu_intensive(n):
    return sum(i * i for i in range(n))

# GIL evita paralelismo real - solo un thread ejecuta a la vez
threads = [
    threading.Thread(target=cpu_intensive, args=(10_000_000,))
    for _ in range(4)
]
for t in threads:
    t.start()
for t in threads:
    t.join()
```

### La Solución

```python
# ✅ CORRECTO - Multiprocessing para CPU-intensive
from multiprocessing import Pool

def cpu_intensive(n):
    return sum(i * i for i in range(n))

# Paralelismo real con procesos separados
with Pool(processes=4) as pool:
    results = pool.map(cpu_intensive, [10_000_000] * 4)

# ✅ CORRECTO - Threads solo para I/O-bound
import requests
from concurrent.futures import ThreadPoolExecutor

def fetch_url(url):
    return requests.get(url).content

# Threads son eficientes para I/O (liberan GIL durante espera)
urls = ['https://api.example.com/data'] * 10
with ThreadPoolExecutor(max_workers=5) as executor:
    results = list(executor.map(fetch_url, urls))
```

---

## 2. Generadores vs Listas

```python
# ❌ INEFICIENTE - Lista completa en memoria
def process_large_file(filename):
    lines = open(filename).readlines()  # Toda la lista en memoria
    for line in lines:
        yield line.strip().upper()

# ✅ CORRECTO - Generador (memoria constante)
def process_large_file(filename):
    with open(filename) as f:
        for line in f:  # Una línea a la vez
            yield line.strip().upper()

# ✅ CORRECTO - Generator expressions
# Lista: consume toda la memoria
squares_list = [x**2 for x in range(1_000_000)]

# Generador: lazy evaluation, memoria constante
squares_gen = (x**2 for x in range(1_000_000))

# ✅ CORRECTO - itertools para operaciones eficientes
from itertools import islice, chain

# Procesar en chunks sin crear lista completa
def chunked(iterable, n):
    iterator = iter(iterable)
    while chunk := list(islice(iterator, n)):
        yield chunk
```

---

## 3. Asyncio y Async/Await

```python
# ❌ INEFICIENTE - Requests síncronos secuenciales
import requests

async def fetch_all_urls(urls):
    results = []
    for url in urls:
        resp = requests.get(url)  # Bloquea cada vez
        results.append(resp.json())
    return results

# ✅ CORRECTO - aiohttp para I/O concurrente
import aiohttp
import asyncio

async def fetch_all_urls(urls):
    async with aiohttp.ClientSession() as session:
        tasks = [fetch_one(session, url) for url in urls]
        return await asyncio.gather(*tasks)

async def fetch_one(session, url):
    async with session.get(url) as resp:
        return await resp.json()

# Ejecución
results = asyncio.run(fetch_all_urls(urls))

# ✅ CORRECTO - async para operaciones de DB
import aiopg

async def get_users_async(user_ids):
    async with aiopg.create_pool(dsn) as pool:
        async with pool.acquire() as conn:
            async with conn.cursor() as cur:
                await cur.execute("SELECT * FROM users WHERE id = ANY(%s)", (user_ids,))
                return await cur.fetchall()
```

---

## 4. Comprensiones vs Loops

```python
# ❌ INEFICIENTE - Loop con append
result = []
for x in range(1000):
    if x % 2 == 0:
        result.append(x * 2)

# ✅ CORRECTO - List comprehension (más rápido)
result = [x * 2 for x in range(1000) if x % 2 == 0]

# ✅ CORRECTO - Dict comprehension
# Loop tradicional
mapping = {}
for key, value in pairs:
    mapping[key] = value

# Dict comprehension (más rápido y claro)
mapping = {key: value for key, value in pairs}

# ✅ CORRECTO - setdefault vs defaultdict
from collections import defaultdict

# ❌ Lento: busca la key múltiples veces
for key, value in items:
    if key not in groups:
        groups[key] = []
    groups[key].append(value)

# ✅ Rápido: defaultdict
groups = defaultdict(list)
for key, value in items:
    groups[key].append(value)
```

---

## 5. Caching con functools.lru_cache

```python
from functools import lru_cache

# ❌ INEFICIENTE - Recalcula fibonacci exponencialmente
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

# ✅ CORRECTO - Memoización automática
@lru_cache(maxsize=128)
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

# ✅ CORRECTO - Cache para operaciones costosas
@lru_cache(maxsize=1024)
def get_user_from_db(user_id: int) -> dict:
    # Costoso: query a base de datos
    return db.query(User).get(user_id).to_dict()

# ✅ CORRECTO - Cache con TTL (time-to-live)
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

## 6. Perfilamiento con cProfile

```python
# ✅ CORRECTO - Identificar cuellos de botella
import cProfile
import pstats

def function_to_profile():
    # Código a analizar
    pass

# Perfilado
profiler = cProfile.Profile()
profiler.enable()
function_to_profile()
profiler.disable()

# Reporte ordenado por tiempo acumulado
stats = pstats.Stats(profiler)
stats.sort_stats('cumulative')
stats.print_stats(20)  # Top 20 funciones

# ✅ CORRECTO - line_profiler para análisis línea por línea
# @profile decorator (ejecutar con kernprof -l -v script.py)
@profile
def slow_function():
    a = [i for i in range(100000)]  # Línea lenta
    b = [i * 2 for i in a]
    return sum(b)

# ✅ CORRECTO - timeit para micro-benchmarks
import timeit

# Comparar dos implementaciones
time_list = timeit.timeit('"-".join(str(n) for n in range(100))', number=10000)
time_gen = timeit.timeit('"-".join(map(str, range(100)))', number=10000)
print(f"List comp: {time_list:.4f}s, Map: {time_gen:.4f}s")
```

---

## 7. N+1 Queries con SQLAlchemy/Django ORM

```python
# ❌ PROHIBIDO - N+1 Queries
# Django
for user in User.objects.all():
    print(user.profile.bio)  # Query adicional por cada usuario

# SQLAlchemy
users = session.query(User).all()
for user in users:
    print(user.orders)  # Query adicional por cada usuario

# ✅ CORRECTO - select_related (JOIN en una query)
# Django
users = User.objects.select_related('profile').all()
for user in users:
    print(user.profile.bio)  # Sin query adicional

# ✅ CORRECTO - prefetch_related (queries separadas + caché)
# Django
users = User.objects.prefetch_related('orders').all()
for user in users:
    for order in user.orders.all():
        print(order.total)

# ✅ CORRECTO - joinedload en SQLAlchemy
from sqlalchemy.orm import joinedload

users = session.query(User).options(
    joinedload(User.profile),
    joinedload(User.orders)
).all()

# ✅ CORRECTO - bulk operations
# ❌ Lento: una query por objeto
for user in users:
    user.active = True
    user.save()

# ✅ Rápido: una sola query UPDATE
User.objects.filter(id__in=[u.id for u in users]).update(active=True)
```

---

## 8. Estructuras de Datos Eficientes

```python
# ❌ INEFICIENTE - Búsqueda O(n) en lista
if item in large_list:  # O(n)
    pass

# ✅ CORRECTO - Búsqueda O(1) en set
large_set = set(large_list)
if item in large_set:  # O(1)
    pass

# ❌ INEFICIENTE - Contar con list.count()
counts = {item: large_list.count(item) for item in set(large_list)}  # O(n²)

# ✅ CORRECTO - Counter O(n)
from collections import Counter
counts = Counter(large_list)  # O(n)

# ✅ CORRECTO - deque para operaciones en extremos
from collections import deque

# ❌ Lista: pop(0) es O(n)
queue = [1, 2, 3]
queue.pop(0)

# ✅ Deque: popleft() es O(1)
queue = deque([1, 2, 3])
queue.popleft()

# ✅ CORRECTO - namedtuple para estructuras ligeras
from collections import namedtuple

# Menor overhead que clase normal
Point = namedtuple('Point', ['x', 'y'])
p = Point(1, 2)
```

---

## 9. String Concatenation

```python
# ❌ INEFICIENTE - O(n²) por inmutabilidad de strings
result = ""
for item in large_list:
    result += str(item)  # Crea nuevo string cada vez

# ✅ CORRECTO - join() O(n)
result = "".join(str(item) for item in large_list)

# ✅ CORRECTO - io.StringIO para concatenación compleja
import io

output = io.StringIO()
for item in large_list:
    output.write(str(item))
    output.write("\n")
result = output.getvalue()

# ✅ CORRECTO - f-strings son más rápidas que % o format
# Más lento
"Hello %s, you have %d messages" % (name, count)
"Hello {}, you have {} messages".format(name, count)

# Más rápido
f"Hello {name}, you have {count} messages"
```

---

## Checklist Python Específico

### Concurrencia
- [ ] Multiprocessing para CPU-bound (GIL)
- [ ] ThreadPoolExecutor para I/O-bound
- [ ] Asyncio para alto throughput de I/O
- [ ] NUNCA usar threads para cálculos intensivos

### Memoria
- [ ] Generadores para grandes volúmenes de datos
- [ ] `__slots__` para clases con muchas instancias
- [ ] Weak references para cachés grandes
- [ ] Streaming en lugar de cargar todo en RAM

### Algoritmos
- [ ] Set/Map para búsquedas frecuentes O(1)
- [ ] Deque para colas (popleft O(1))
- [ ] Counter para conteo de frecuencias
- [ ] Sort con key en lugar de cmp

### Base de Datos
- [ ] select_related para ForeignKey (1 query)
- [ ] prefetch_related para ManyToMany (2 queries)
- [ ] Bulk operations en lugar de loops
- [ ] Índices en columnas de búsqueda frecuente

### Caching
- [ ] @lru_cache para funciones puras
- [ ] Cachear resultados de queries frecuentes
- [ ] Invalidación apropiada del caché

### Profiling
- [ ] cProfile para identificar cuellos de botella
- [ ] timeit para comparar implementaciones
- [ ] memory_profiler para uso de memoria

---

## Referencias

- [Python Performance Tips](https://wiki.python.org/moin/PythonSpeed/PerformanceTips)
- [High Performance Python](https://www.oreilly.com/library/view/high-performance-python/9781449361539/)
- [Django Database Optimization](https://docs.djangoproject.com/en/stable/topics/db/optimization/)
- [SQLAlchemy Performance](https://docs.sqlalchemy.org/en/20/faq/performance.html)
