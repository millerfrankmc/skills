# Performance Specific - TypeScript/JavaScript

> Guide for performance optimizations in TypeScript/JavaScript code. NEVER sacrifice obvious performance for "cleaner code".

---

## 1. Event Loop and Async/Await

### The Problem

```typescript
// ❌ INEFFICIENT - Blocking the event loop
function calculateSync(data: number[]): number[] {
    return data.map(n => {
        // CPU-intensive operation blocks entire server
        let result = 0;
        for (let i = 0; i < n; i++) {
            result += Math.sqrt(i);
        }
        return result;
    });
}

// ❌ INEFFICIENT - Saturated microtask queue
async function processAll(items: Item[]) {
    for (const item of items) {
        await process(item); // Each await generates microtask
    }
}
```

### The Solution

```typescript
// ✅ CORRECT - setImmediate to yield event loop
function calculateNonBlocking(data: number[]): Promise<number[]> {
    return new Promise((resolve) => {
        const results: number[] = [];
        let index = 0;

        function processChunk() {
            const chunk = 1000;
            for (let i = 0; i < chunk && index < data.length; i++, index++) {
                // Process element
                results.push(heavyCalculation(data[index]));
            }

            if (index < data.length) {
                setImmediate(processChunk); // Yield event loop
            } else {
                resolve(results);
            }
        }

        processChunk();
    });
}

// ✅ CORRECT - Worker threads for CPU-intensive
import { Worker, isMainThread, parentPort, workerData } from 'worker_threads';

if (isMainThread) {
    // Main thread
    const worker = new Worker(__filename, {
        workerData: largeDataset
    });

    worker.on('message', (result) => {
        console.log('Result:', result);
    });
} else {
    // Worker thread - doesn't block main thread
    const result = heavyCalculation(workerData);
    parentPort?.postMessage(result);
}
```

---

## 2. Promise.all vs Serial Loops

```typescript
// ❌ INEFFICIENT - Unnecessary serialization
async function fetchUsers(userIds: string[]): Promise<User[]> {
    const users: User[] = [];
    for (const id of userIds) {
        const user = await db.users.findById(id); // Waits for each one
        users.push(user);
    }
    return users;
}

// ✅ CORRECT - Parallelism with Promise.all
async function fetchUsers(userIds: string[]): Promise<User[]> {
    const promises = userIds.map(id => db.users.findById(id));
    return Promise.all(promises); // All in parallel
}

// ✅ CORRECT - Concurrency limit to avoid saturation
import pLimit from 'p-limit';

const limit = pLimit(10); // Maximum 10 concurrent

async function fetchWithLimit(userIds: string[]): Promise<User[]> {
    const promises = userIds.map(id =>
        limit(() => db.users.findById(id))
    );
    return Promise.all(promises);
}

// ✅ CORRECT - Promise.allSettled for error handling
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
// ❌ INEFFICIENT - O(n) search in array
const allowedIds = ['a', 'b', 'c', 'd', ...]; // 1000+ elements

function isAllowed(id: string): boolean {
    return allowedIds.includes(id); // O(n)
}

items.filter(item => isAllowed(item.id)); // O(n * m)

// ✅ CORRECT - O(1) search with Set
const allowedSet = new Set(allowedIds);

function isAllowed(id: string): boolean {
    return allowedSet.has(id); // O(1)
}

items.filter(item => isAllowed(item.id)); // O(n)

// ✅ CORRECT - Map for ID lookup
// ❌ Array: find is O(n)
const user = users.find(u => u.id === userId);

// ✅ Map: get is O(1)
const userMap = new Map(users.map(u => [u.id, u]));
const user = userMap.get(userId);

// ✅ CORRECT - Set for deduplication
// ❌ Slow: filter + indexOf
const unique = items.filter((item, index) =>
    items.indexOf(item) === index
); // O(n²)

// ✅ Fast: Set
const unique = [...new Set(items)]; // O(n)
```

---

## 4. Memoization with Closures

```typescript
// ✅ CORRECT - Simple memoization
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

// Usage
const fibonacci = memoize((n: number): number => {
    if (n < 2) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
});

// ✅ CORRECT - LRU Cache with size limit
class LRUCache<K, V> {
    private cache = new Map<K, V>();

    constructor(private maxSize: number) {}

    get(key: K): V | undefined {
        const value = this.cache.get(key);
        if (value !== undefined) {
            // Move to end (most recent)
            this.cache.delete(key);
            this.cache.set(key, value);
        }
        return value;
    }

    set(key: K, value: V): void {
        if (this.cache.size >= this.maxSize) {
            // Remove oldest
            const firstKey = this.cache.keys().next().value;
            this.cache.delete(firstKey);
        }
        this.cache.set(key, value);
    }
}

// ✅ CORRECT - Async function memoization
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

// Usage for DB queries
const getUserCached = memoizeAsync(
    (userId: string) => db.users.findById(userId),
    5000 // 5 seconds TTL
);
```

---

## 5. Lazy Module Loading

```typescript
// ❌ INEFFICIENT - Load everything at startup
import { HeavyLibrary } from 'heavy-library';
import { AnotherHeavy } from 'another-heavy';

const lib = new HeavyLibrary(); // Immediate load, memory used

// ✅ CORRECT - Dynamic import (lazy loading)
async function processWithHeavyLib(data: Data) {
    const { HeavyLibrary } = await import('heavy-library');
    const lib = new HeavyLibrary();
    return lib.process(data);
}

// ✅ CORRECT - Lazy initialization with closure
function createLazyLoader<T>(factory: () => T): () => T {
    let instance: T | undefined;

    return () => {
        if (instance === undefined) {
            instance = factory();
        }
        return instance;
    };
}

// Usage
const getConfig = createLazyLoader(() => {
    console.log('Loading config...');
    return loadConfigFromFile();
});

// Config doesn't load until needed
const config = getConfig();

// ✅ CORRECT - React.lazy for code splitting
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

## 6. Worker Threads for CPU-intensive

```typescript
// ✅ CORRECT - Worker pool
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
            // Process next task
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

// Usage
const pool = new WorkerPool('./image-processor.worker.js', 4);
const results = await Promise.all(
    images.map(img => pool.execute({ image: img, operation: 'resize' }))
);
```

---

## 7. N+1 Queries with Prisma/TypeORM

```typescript
// ❌ PROHIBITED - N+1 Queries
// Prisma
const users = await prisma.user.findMany();
for (const user of users) {
    // Additional query per user!
    const posts = await prisma.post.findMany({
        where: { authorId: user.id }
    });
}

// TypeORM
const users = await userRepo.find();
for (const user of users) {
    const posts = await postRepo.find({ where: { authorId: user.id } });
}

// ✅ CORRECT - Prisma include (eager loading)
const users = await prisma.user.findMany({
    include: {
        posts: true,      // JOIN in one query
        profile: true,    // Related SELECT
    }
});

// ✅ CORRECT - TypeORM relations
const users = await userRepo.find({
    relations: ['posts', 'profile']
});

// ✅ CORRECT - Raw query when ORM is insufficient
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

// ✅ CORRECT - Batch loading with DataLoader
import DataLoader from 'dataloader';

const postLoader = new DataLoader(async (userIds: string[]) => {
    const posts = await prisma.post.findMany({
        where: { authorId: { in: userIds } }
    });
    // Group by userId
    const postsByUser = groupBy(posts, 'authorId');
    return userIds.map(id => postsByUser.get(id) || []);
});

// Now this makes a single query
for (const user of users) {
    const posts = await postLoader.load(user.id); // Batched!
}
```

---

## TypeScript Specific Checklist

### Concurrency
- [ ] Promise.all for independent operations
- [ ] p-limit to control concurrency
- [ ] Worker threads for CPU-intensive
- [ ] setImmediate to yield event loop

### Data Structures
- [ ] Set.has() instead of array.includes()
- [ ] Map.get() instead of array.find()
- [ ] new Set() for deduplication

### Caching
- [ ] Memoization for expensive calculations
- [ ] DataLoader for N+1 queries
- [ ] LRU cache with TTL

### Modules
- [ ] Dynamic imports for lazy loading
- [ ] React.lazy for code splitting
- [ ] Lazy initialization

### Database
- [ ] include/relations for eager loading
- [ ] DataLoader for GraphQL resolvers
- [ ] Raw query for complex queries

### Strings
- [ ] Array.join() instead of += in loops
- [ ] Efficient template literals

---

## References

- [Node.js Performance](https://nodejs.org/en/docs/guides/simple-profiling/)
- [Worker Threads](https://nodejs.org/api/worker_threads.html)
- [Prisma Performance](https://www.prisma.io/docs/guides/performance-and-optimization)
- [TypeORM Performance](https://typeorm.io/#/performance/)
