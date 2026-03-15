# Rendimiento Específico - TypeScript/JavaScript

> Guía de optimizaciones de rendimiento para código TypeScript/JavaScript. NUNCA sacrificar rendimiento obvio por "código más limpio".

---

## 1. Event Loop y Async/Await

### El Problema

```typescript
// ❌ INEFICIENTE - Bloquear el event loop
function calculateSync(data: number[]): number[] {
    return data.map(n => {
        // Operación CPU-intensive bloquea todo el servidor
        let result = 0;
        for (let i = 0; i < n; i++) {
            result += Math.sqrt(i);
        }
        return result;
    });
}

// ❌ INEFICIENTE - Microtask queue saturada
async function processAll(items: Item[]) {
    for (const item of items) {
        await process(item); // Cada await genera microtask
    }
}
```

### La Solución

```typescript
// ✅ CORRECTO - setImmediate para ceder el event loop
function calculateNonBlocking(data: number[]): Promise<number[]> {
    return new Promise((resolve) => {
        const results: number[] = [];
        let index = 0;

        function processChunk() {
            const chunk = 1000;
            for (let i = 0; i < chunk && index < data.length; i++, index++) {
                // Procesar elemento
                results.push(heavyCalculation(data[index]));
            }

            if (index < data.length) {
                setImmediate(processChunk); // Cedemos el event loop
            } else {
                resolve(results);
            }
        }

        processChunk();
    });
}

// ✅ CORRECTO - Worker threads para CPU-intensive
import { Worker, isMainThread, parentPort, workerData } from 'worker_threads';

if (isMainThread) {
    // Main thread
    const worker = new Worker(__filename, {
        workerData: largeDataset
    });

    worker.on('message', (result) => {
        console.log('Resultado:', result);
    });
} else {
    // Worker thread - no bloquea el main thread
    const result = heavyCalculation(workerData);
    parentPort?.postMessage(result);
}
```

---

## 2. Promise.all vs Loops Seriales

```typescript
// ❌ INEFICIENTE - Serialización innecesaria
async function fetchUsers(userIds: string[]): Promise<User[]> {
    const users: User[] = [];
    for (const id of userIds) {
        const user = await db.users.findById(id); // Espera cada uno
        users.push(user);
    }
    return users;
}

// ✅ CORRECTO - Paralelismo con Promise.all
async function fetchUsers(userIds: string[]): Promise<User[]> {
    const promises = userIds.map(id => db.users.findById(id));
    return Promise.all(promises); // Todas en paralelo
}

// ✅ CORRECTO - Límite de concurrencia para evitar saturación
import pLimit from 'p-limit';

const limit = pLimit(10); // Máximo 10 concurrentes

async function fetchWithLimit(userIds: string[]): Promise<User[]> {
    const promises = userIds.map(id =>
        limit(() => db.users.findById(id))
    );
    return Promise.all(promises);
}

// ✅ CORRECTO - Promise.allSettled para manejo de errores
async function fetchWithErrorHandling(userIds: string[]) {
    const results = await Promise.allSettled(
        userIds.map(id => db.users.findById(id))
    );

    return results.map((result, index) => {
        if (result.status === 'fulfilled') {
            return { id: userIds[index], user: result.value };
        } else {
            return { id: userIds[index], error: result.reason };
        }
    });
}
```

---

## 3. Map/Set vs Arrays

```typescript
// ❌ INEFICIENTE - Búsqueda O(n) en array
const allowedIds = ['a', 'b', 'c', 'd', ...]; // 1000+ elementos

function isAllowed(id: string): boolean {
    return allowedIds.includes(id); // O(n)
}

items.filter(item => isAllowed(item.id)); // O(n * m)

// ✅ CORRECTO - Búsqueda O(1) con Set
const allowedSet = new Set(allowedIds);

function isAllowed(id: string): boolean {
    return allowedSet.has(id); // O(1)
}

items.filter(item => isAllowed(item.id)); // O(n)

// ✅ CORRECTO - Map para lookup por ID
// ❌ Array: find es O(n)
const user = users.find(u => u.id === userId);

// ✅ Map: get es O(1)
const userMap = new Map(users.map(u => [u.id, u]));
const user = userMap.get(userId);

// ✅ CORRECTO - Set para deduplicación
// ❌ Lento: filter + indexOf
const unique = items.filter((item, index) =>
    items.indexOf(item) === index
); // O(n²)

// ✅ Rápido: Set
const unique = [...new Set(items)]; // O(n)
```

---

## 4. Memoization con Closures

```typescript
// ✅ CORRECTO - Memoización simple
function memoize<T, R>(fn: (arg: T) => R): (arg: T) => R {
    const cache = new Map<T, R>();

    return (arg: T) => {
        if (cache.has(arg)) {
            return cache.get(arg)!;
        }

        const result = fn(arg);
        cache.set(arg, result);
        return result;
    };
}

// Uso
const fibonacci = memoize((n: number): number => {
    if (n < 2) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
});

// ✅ CORRECTO - LRU Cache con límite de tamaño
class LRUCache<K, V> {
    private cache = new Map<K, V>();

    constructor(private maxSize: number) {}

    get(key: K): V | undefined {
        const value = this.cache.get(key);
        if (value !== undefined) {
            // Mover al final (más reciente)
            this.cache.delete(key);
            this.cache.set(key, value);
        }
        return value;
    }

    set(key: K, value: V): void {
        if (this.cache.size >= this.maxSize) {
            // Eliminar el más antiguo
            const firstKey = this.cache.keys().next().value;
            this.cache.delete(firstKey);
        }
        this.cache.set(key, value);
    }
}

// ✅ CORRECTO - Memoización de funciones asíncronas
function memoizeAsync<T, R>(
    fn: (arg: T) => Promise<R>,
    ttl: number = 60000
): (arg: T) => Promise<R> {
    const cache = new Map<T, { value: R; timestamp: number }>();

    return async (arg: T) => {
        const cached = cache.get(arg);
        const now = Date.now();

        if (cached && now - cached.timestamp < ttl) {
            return cached.value;
        }

        const result = await fn(arg);
        cache.set(arg, { value: result, timestamp: now });
        return result;
    };
}

// Uso para queries a DB
const getUserCached = memoizeAsync(
    (userId: string) => db.users.findById(userId),
    5000 // 5 segundos TTL
);
```

---

## 5. Lazy Loading de Módulos

```typescript
// ❌ INEFICIENTE - Carga todo al inicio
import { HeavyLibrary } from 'heavy-library';
import { AnotherHeavy } from 'another-heavy';

const lib = new HeavyLibrary(); // Carga inmediata, memoria usada

// ✅ CORRECTO - Dynamic import (lazy loading)
async function processWithHeavyLib(data: Data) {
    const { HeavyLibrary } = await import('heavy-library');
    const lib = new HeavyLibrary();
    return lib.process(data);
}

// ✅ CORRECTO - Lazy initialization con closure
function createLazyLoader<T>(factory: () => T): () => T {
    let instance: T | undefined;

    return () => {
        if (instance === undefined) {
            instance = factory();
        }
        return instance;
    };
}

// Uso
const getConfig = createLazyLoader(() => {
    console.log('Loading config...');
    return loadConfigFromFile();
});

// Config no se carga hasta que se necesite
const config = getConfig();

// ✅ CORRECTO - React.lazy para code splitting
import React, { Suspense, lazy } from 'react';

const HeavyComponent = lazy(() => import('./HeavyComponent'));

function App() {
    return (
        <Suspense fallback={<Loading />}>
            <HeavyComponent />
        </Suspense>
    );
}
```

---

## 6. Worker Threads para CPU-intensive

```typescript
// ✅ CORRECTO - Pool de workers
import { Worker } from 'worker_threads';
import os from 'os';

class WorkerPool {
    private workers: Worker[] = [];
    private queue: Array<{ task: any; resolve: Function; reject: Function }> = [];
    private activeWorkers = 0;

    constructor(
        private workerScript: string,
        private poolSize = os.cpus().length
    ) {
        for (let i = 0; i < poolSize; i++) {
            this.addWorker();
        }
    }

    private addWorker() {
        const worker = new Worker(this.workerScript);

        worker.on('message', (result) => {
            this.activeWorkers--;
            // Procesar siguiente tarea
            this.processQueue();
        });

        this.workers.push(worker);
    }

    private processQueue() {
        if (this.queue.length === 0 || this.activeWorkers >= this.poolSize) {
            return;
        }

        const { task, resolve, reject } = this.queue.shift()!;
        const worker = this.workers[this.activeWorkers];
        this.activeWorkers++;

        worker.once('message', resolve);
        worker.once('error', reject);
        worker.postMessage(task);
    }

    execute(task: any): Promise<any> {
        return new Promise((resolve, reject) => {
            this.queue.push({ task, resolve, reject });
            this.processQueue();
        });
    }

    terminate() {
        return Promise.all(this.workers.map(w => w.terminate()));
    }
}

// Uso
const pool = new WorkerPool('./image-processor.worker.js', 4);
const results = await Promise.all(
    images.map(img => pool.execute({ image: img, operation: 'resize' }))
);
```

---

## 7. N+1 Queries con Prisma/TypeORM

```typescript
// ❌ PROHIBIDO - N+1 Queries
// Prisma
const users = await prisma.user.findMany();
for (const user of users) {
    // Query adicional por cada usuario!
    const posts = await prisma.post.findMany({
        where: { authorId: user.id }
    });
}

// TypeORM
const users = await userRepo.find();
for (const user of users) {
    const posts = await postRepo.find({ where: { authorId: user.id } });
}

// ✅ CORRECTO - Prisma include (eager loading)
const users = await prisma.user.findMany({
    include: {
        posts: true,      // JOIN en una query
        profile: true,    // SELECT relacionado
    }
});

// ✅ CORRECTO - TypeORM relations
const users = await userRepo.find({
    relations: ['posts', 'profile']
});

// ✅ CORRECTO - Query raw cuando ORM no es suficiente
// Prisma
const usersWithPostCount = await prisma.$queryRaw`
    SELECT u.*, COUNT(p.id) as post_count
    FROM "User" u
    LEFT JOIN "Post" p ON p."authorId" = u.id
    GROUP BY u.id
`;

// TypeORM
const usersWithPostCount = await dataSource.query(`
    SELECT u.*, COUNT(p.id) as post_count
    FROM users u
    LEFT JOIN posts p ON p.author_id = u.id
    GROUP BY u.id
`);

// ✅ CORRECTO - Batch loading con DataLoader
import DataLoader from 'dataloader';

const postLoader = new DataLoader(async (userIds: string[]) => {
    const posts = await prisma.post.findMany({
        where: { authorId: { in: userIds } }
    });
    // Agrupar por userId
    const postsByUser = groupBy(posts, 'authorId');
    return userIds.map(id => postsByUser.get(id) || []);
});

// Ahora esto hace una sola query
for (const user of users) {
    const posts = await postLoader.load(user.id); // Batched!
}
```

---

## Checklist TypeScript Específico

### Concurrencia
- [ ] Promise.all para operaciones independientes
- [ ] p-limit para controlar concurrencia
- [ ] Worker threads para CPU-intensive
- [ ] setImmediate para ceder event loop

### Estructuras de Datos
- [ ] Set.has() en lugar de array.includes()
- [ ] Map.get() en lugar de array.find()
- [ ] new Set() para deduplicación

### Caching
- [ ] Memoización para cálculos costosos
- [ ] DataLoader para N+1 queries
- [ ] LRU cache con TTL

### Módulos
- [ ] Dynamic imports para lazy loading
- [ ] React.lazy para code splitting
- [ ] Lazy initialization

### Base de Datos
- [ ] include/relations para eager loading
- [ ] DataLoader para resolvers GraphQL
- [ ] Query raw para queries complejas

### Strings
- [ ] Array.join() en lugar de += en loops
- [ ] Template literals eficientes

---

## Referencias

- [Node.js Performance](https://nodejs.org/en/docs/guides/simple-profiling/)
- [Worker Threads](https://nodejs.org/api/worker_threads.html)
- [Prisma Performance](https://www.prisma.io/docs/guides/performance-and-optimization)
- [TypeORM Performance](https://typeorm.io/#/performance/)
