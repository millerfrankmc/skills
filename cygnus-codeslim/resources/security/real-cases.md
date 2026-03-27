# Real Cases of Security Accidentally Removed

> This file documents specific cases where the KISS-DRY-YAGNI skill removed security code by mistake. Use it as a reference to understand which protections are easy to confuse with "just in case" code.

---

## Protection Index

### 📚 Documentation by Language

| Language | Security | Performance |
|----------|----------|-------------|
| **Python** | [python-security.md](./python-security.md) | [python-performance.md](../performance/python-performance.md) |
| **TypeScript** | [typescript-security.md](./typescript-security.md) | [typescript-performance.md](../performance/typescript-performance.md) |
| **Go** | [go-security.md](./go-security.md) | [go-performance.md](../performance/go-performance.md) |
| **Kotlin** | [kotlin-security.md](./kotlin-security.md) | [kotlin-performance.md](../performance/kotlin-performance.md) |
| **Rust** | [rust-security.md](./rust-security.md) | [rust-performance.md](../performance/rust-performance.md) |

### 🔒 Cases Documented in This File

| Case | Category | Description |
|------|----------|-------------|
| [Case 1: Graceful Shutdown](#case-1-graceful-shutdown) | Process Protection | SIGTERM handling for orderly shutdown |
| [Case 2: Non-Root User in Docker](#case-2-non-root-user-in-docker) | Infrastructure Protection | Security in containers |
| [Case 3: Circuit Breaker](#case-3-circuit-breaker-removed) | Resilience | Prevention of cascading failures |
| [Case 4: Rate Limiting](#case-4-rate-limiting-removed) | Abuse Protection | Brute force prevention |
| [Case 5: Multiple Validation](#case-5-multiple-validation-removed) | Defense in Depth | Upload validation |
| [Case 6: Resource Limits](#case-6-resource-limits-removed) | Infrastructure Protection | Limits in Kubernetes |

### ⚡ Performance Optimizations by Language

Consult the specific performance files for each language:

- **Python**: GIL and threading vs multiprocessing, generators vs lists, asyncio, N+1 queries
- **TypeScript**: Event loop and async/await, Promise.all vs serial loops, Worker threads
- **Go**: Goroutines and channels, sync.Pool, strings.Builder, slice pre-allocation
- **Kotlin**: Coroutines vs threads, lazy initialization, inline functions, Flow
- **Rust**: Zero-cost abstractions, iterators, SIMD, async/await with Tokio

---

---

## Case 1: Graceful Shutdown

**Date:** 2026-03-13
**Context:** Node.js service with signal handling
**File:** `src/server.ts`

### Code Removed

```javascript
process.on('SIGTERM', async () => {
  logger.info('Received SIGTERM, starting graceful shutdown...');

  // Stop accepting new connections
  server.close(async () => {
    logger.info('HTTP server closed');

    try {
      // Close database connections
      await db.close();
      logger.info('Database connections closed');

      // Close cache connections
      await redis.disconnect();
      logger.info('Cache connections closed');

      logger.info('Graceful shutdown complete');
      process.exit(0);
    } catch (error) {
      logger.error('Error during shutdown:', error);
      process.exit(1);
    }
  });
});
```

### Incorrect Removal Reason
> "This code is 'just in case'. The operating system already handles process termination automatically. Also, the code is too long (20+ lines) for a function that just closes connections."

### Real Impact

1. **Data corruption:** Requests in progress were interrupted mid-processing
2. **Hung connections:** Database connections were left in `IDLE IN TRANSACTION` state
3. **Transaction loss:** 3 financial transactions were left in inconsistent state
4. **Slow reconnection:** The next deployment took 5 extra minutes cleaning up ghost connections

### Why It's Security (Not YAGNI)

```
┌─────────────────────────────────────────────────────────────┐
│  Without graceful shutdown      With graceful shutdown      │
├─────────────────────────────────────────────────────────────┤
│  1. SIGTERM received            1. SIGTERM received         │
│  2. Process dies IMMEDIATELY    2. Signal captured          │
│  3. Requests aborted            3. Stop accepting new       │
│  4. DB in inconsistent state    4. Wait for active requests │
│  5. Corrupt data                5. Close connections        │
│                                 6. Exit cleanly             │
└─────────────────────────────────────────────────────────────┘
```

### Classification

- **Category:** Process Protection
- **Subcategory:** Graceful Shutdown
- **Keywords:** `SIGTERM`, `SIGINT`, `process.on`, `server.close()`, `cleanup`

---

## Case 2: Non-Root User in Docker

**Date:** 2026-03-13
**Context:** Dockerfile for Python/Flask application
**File:** `Dockerfile`

### Code Removed

```dockerfile
FROM python:3.11-slim

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Create non-root user to run the application
RUN groupadd -r appgroup && useradd -r -g appgroup appuser

# Create working directory
WORKDIR /app

# Copy requirements first (for layer cache)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Change owner of code to non-root user
RUN chown -R appuser:appgroup /app

# Switch to non-root user
USER appuser

# Expose port
EXPOSE 5000

# Start command
CMD ["python", "app.py"]
```

### Incorrect Removal Reason
> "Creating a user is unnecessary complication. Docker containers are already isolated from the host. According to YAGNI, if there's no evidence that someone escapes containers, we don't need this protection. Also, the container needs root to install dependencies."

### Real Impact

1. **Privilege escalation:** If a vulnerability in the app is exploited (e.g., RCE via deserialization), the attacker has **root inside the container**
2. **Container escape:** With root + kernel vulnerability (very common), the attacker gets **root on the host**
3. **Infrastructure attack:** The attacker can:
   - Access other containers
   - Mount host volumes
   - Modify system configurations
   - Install persistent backdoors

### Risk Demonstration

```bash
# Without USER directive (running as root)
$ docker exec vulnerable_app whoami
root

# With an RCE vulnerability, the attacker can:
root@container:~# mount /dev/sda1 /mnt/host
root@container:~# cat /mnt/host/etc/shadow
root@container:~# echo "backdoor" >> /mnt/host/root/.bashrc
```

### Why It's Security (Not YAGNI)

```
┌────────────────────────────────────────────────────────────────┐
│           Defense in Depth                                     │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│   Layer 1: Container isolation (not guaranteed)               │
│   ├─ Namespace isolation                                      │
│   ├─ Cgroups                                                  │
│   └─ Capabilities                                             │
│                                                                │
│   Layer 2: Non-root user (this protection)                    │
│   ├─ If attacker escapes, they don't have root on host       │
│   └─ Limits impact of RCE inside the container               │
│                                                                │
│   Layer 3: Seccomp, AppArmor (additional protection)          │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Classification

- **Category:** Infrastructure Protection
- **Subcategory:** Secure Container
- **Keywords:** `USER`, `useradd`, `adduser`, `runAsNonRoot`, `securityContext`

---

## Case 3: Circuit Breaker Removed

**Date:** 2025-11-20
**Context:** Payment microservice with bank API calls
**File:** `src/services/bankApi.ts`

### Code Removed

```typescript
class BankApiCircuitBreaker {
  private failureCount = 0;
  private lastFailureTime: number | null = null;
  private state: 'CLOSED' | 'OPEN' | 'HALF_OPEN' = 'CLOSED';

  private readonly FAILURE_THRESHOLD = 5;
  private readonly TIMEOUT = 60000; // 1 minute

  async execute<T>(operation: () => Promise<T>): Promise<T> {
    if (this.state === 'OPEN') {
      if (Date.now() - (this.lastFailureTime || 0) > this.TIMEOUT) {
        this.state = 'HALF_OPEN';
      } else {
        throw new CircuitBreakerOpenError('Bank API is down');
      }
    }

    try {
      const result = await operation();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess() {
    this.failureCount = 0;
    this.state = 'CLOSED';
  }

  private onFailure() {
    this.failureCount++;
    this.lastFailureTime = Date.now();

    if (this.failureCount >= this.FAILURE_THRESHOLD) {
      this.state = 'OPEN';
      logger.error('Circuit breaker opened for Bank API');
    }
  }
}
```

### Incorrect Removal Reason
> "This code is too complex (40+ lines) for a simple error counter. Normal error handling is sufficient. Also, if the bank API fails, we want to know immediately, not 'open a circuit'."

### Real Impact

1. **Cascading failures:** When the bank API had 30s latency, ALL services that depended on it also degraded
2. **Massive timeout:** Each request waited 30s before failing, exhausting the connection pool
3. **Denial of Service:** The service stopped responding to new requests for 15 minutes
4. **Revenue loss:** 200 payment transactions not processed

### Why It's Security/Resilience (Not YAGNI)

```
┌────────────────────────────────────────────────────────────────┐
│  Without Circuit Breaker        With Circuit Breaker          │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  Request 1: timeout 30s         Request 1: timeout 30s        │
│  Request 2: timeout 30s         Request 2: timeout 30s        │
│  Request 3: timeout 30s         Request 3: timeout 30s        │
│  ...                            Request 4: timeout 30s        │
│  Request 100: timeout 30s       Request 5: timeout 30s        │
│                                 ────────────────────────      │
│  Total: 3000s of waiting        CIRCUIT BREAKER OPEN          │
│  Connection pool exhausted      Request 6: IMMEDIATE failure  │
│  Service unavailable            (no waiting)                  │
│                                 Service remains available     │
│                                 for other operations          │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Classification

- **Category:** Resilience
- **Subcategory:** Circuit Breaker
- **Keywords:** `CircuitBreaker`, `failureCount`, `threshold`, `OPEN`, `CLOSED`

---

## Case 4: Rate Limiting Removed

**Date:** 2025-09-15
**Context:** Web application login endpoint
**File:** `src/routes/auth.ts`

### Code Removed

```typescript
import rateLimit from 'express-rate-limit';

const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 5, // maximum 5 attempts
  skipSuccessfulRequests: true,
  standardHeaders: true,
  legacyHeaders: false,
  handler: (req, res) => {
    res.status(429).json({
      error: 'Too many login attempts',
      retryAfter: Math.ceil(req.rateLimit.resetTime / 1000)
    });
  }
});

// Apply only to login endpoint
app.post('/auth/login', loginLimiter, async (req, res) => {
  // ... login logic
});
```

### Incorrect Removal Reason
> "Rate limiting should be done at the infrastructure level (Nginx, CloudFlare), not in code. This violates YAGNI because we already have protection at the edge. Also, the code is complex (14 lines) for something the proxy should handle."

### Real Impact

1. **Brute force attack:** 100,000 login attempts against administrator accounts
2. **Account compromise:** 3 admin accounts were compromised using common passwords
3. **Unauthorized access:** The attacker accessed the admin panel
4. **Data exfiltration:** Data from 10,000 users was exported

### Why It's Security (Not YAGNI)

```
┌────────────────────────────────────────────────────────────────┐
│  Defense in Depth for Authentication                           │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  Layer 1: Edge (CloudFlare/Nginx)                             │
│  ├─ DDoS protection                                           │
│  ├─ Rate limiting by IP                                       │
│  └─ Can be bypassed with IP rotation                         │
│                                                                │
│  Layer 2: Application (this rate limiting)                    │
│  ├─ Rate limiting by account (not just IP)                   │
│  ├─ Protection against credential stuffing                    │
│  └─ Logging of suspicious attempts                            │
│                                                                │
│  Layer 3: Database                                            │
│  ├─ Account lockout after N attempts                          │
│  └─ User notification                                         │
│                                                                │
│  If you remove Layer 2 → attacker can attack from multiple   │
│  IPs without being detected                                   │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Classification

- **Category:** Abuse Protection
- **Subcategory:** Rate Limiting
- **Keywords:** `rateLimit`, `max`, `windowMs`, `handler`

---

## Case 5: Multiple Validation Removed

**Date:** 2025-07-10
**Context:** File upload in document application
**File:** `src/services/fileUpload.ts`

### Code Removed

```typescript
async function validateAndProcessFile(file: UploadedFile) {
  // Layer 1: Validate MIME type from header
  const allowedMimeTypes = ['application/pdf', 'image/jpeg', 'image/png'];
  if (!allowedMimeTypes.includes(file.mimetype)) {
    throw new ValidationError('Invalid MIME type');
  }

  // Layer 2: Validate file extension
  const ext = path.extname(file.originalname).toLowerCase();
  const allowedExtensions = ['.pdf', '.jpg', '.jpeg', '.png'];
  if (!allowedExtensions.includes(ext)) {
    throw new ValidationError('Invalid file extension');
  }

  // Layer 3: Validate magic bytes (actual file content)
  const magicBytes = await readFirstBytes(file.path, 8);
  if (!isValidMagicBytes(magicBytes, file.mimetype)) {
    throw new ValidationError('File content does not match declared type');
  }

  // Layer 4: Validate maximum size
  const MAX_SIZE = 10 * 1024 * 1024; // 10MB
  if (file.size > MAX_SIZE) {
    throw new ValidationError('File too large');
  }

  // Layer 5: Scan with antivirus (ClamAV)
  const scanResult = await clamav.scan(file.path);
  if (scanResult.isInfected) {
    throw new ValidationError('File contains malware');
  }

  return processFile(file);
}
```

### Incorrect Removal Reason
> "This function violates DRY - it has 5 different validations for the 'same' file. We should consolidate into a single `validateFile()` function. Also, validating MIME type AND extension AND magic bytes is redundant - if the MIME type is correct, the file is valid."

### Real Impact

1. **Security bypass:** An attacker uploaded a `.exe` file renamed to `.pdf`
2. **Malware execution:** The file exploited a vulnerability in the PDF viewer
3. **Server compromise:** A cryptominer was installed
4. **Lateral movement:** The attacker accessed the internal network

### Why It's Security (Not DRY)

```
┌────────────────────────────────────────────────────────────────┐
│  Defense in Depth - Each layer protects against different     │
│  attack vector                                                │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  Layer 1: MIME type                                            │
│  └─ Protects against: Browser executing file as script       │
│                                                                │
│  Layer 2: File extension                                       │
│  └─ Protects against: Type confusion in file system          │
│                                                                │
│  Layer 3: Magic bytes                                          │
│  └─ Protects against: MIME spoofing (false header)            │
│                                                                │
│  Layer 4: Size                                                 │
│  └─ Protects against: DoS by disk full                        │
│                                                                │
│  Layer 5: Antivirus                                            │
│  └─ Protects against: Known malware                           │
│                                                                │
│  If you consolidate into a single validation → easy bypass   │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Classification

- **Category:** Defense in Depth
- **Subcategory:** Multiple Validation
- **Keywords:** `mimetype`, `extension`, `magic bytes`, `validation`

---

## Case 6: Resource Limits Removed

**Date:** 2025-05-22
**Context:** Deployment in Kubernetes
**File:** `k8s/deployment.yaml`

### Code Removed

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: api
          image: api:latest
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          env:
            - name: NODE_OPTIONS
              value: "--max-old-space-size=450"
```

### Incorrect Removal Reason
> "Resource limits are premature optimization. Kubernetes will allocate resources dynamically as needed. Also, these values are arbitrary (why 512Mi and not 1Gi?). When we need more resources, we'll adjust them."

### Real Impact

1. **Memory leak in production:** A bug caused memory consumption to grow indefinitely
2. **OOM Kill cascade:** Without limits, the pod consumed 8GB of RAM and was killed
3. **Node pressure:** The entire node ran out of memory
4. **Pod eviction:** Kubernetes started killing other pods to free memory
5. **Total degradation:** The entire application was unusable for 30 minutes

### Why It's Security/Stability (Not Premature Optimization)

```
┌────────────────────────────────────────────────────────────────┐
│  Without Resource Limits        With Resource Limits          │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  Memory leak occurs             Memory leak occurs            │
│  ├─ Consumption grows unbounded ├─ Reaches limit (512Mi)      │
│  ├─ Pod uses 8GB RAM            ├─ Pod is OOM Killed          │
│  ├─ Node out of memory          ├─ Kubernetes restarts pod    │
│  ├─ Other pods evicted          ├─ System remains stable      │
│  └─ Total degradation           └─ Single pod affected        │
│                                                                │
│  Limit = Fail fast + Damage containment                       │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Classification

- **Category:** Infrastructure Protection
- **Subcategory:** Resource Limits
- **Keywords:** `resources`, `limits`, `requests`, `memory`, `cpu`

---

## Common Error Patterns

### Pattern 1: "The System Already Handles It"
> "The OS/Container/Kubernetes already handles that"

**Examples:**
- Graceful shutdown → "The OS kills the process"
- Non-root user → "Docker already isolates"
- Rate limiting → "Nginx already does it"

**Reality:** Defense in depth - each layer is necessary

### Pattern 2: "It Has Never Failed"
> "We've been running for 2 years without this being a problem"

**Examples:**
- Circuit breaker → "We've never had cascading failures"
- Resource limits → "We've never had memory leaks"

**Reality:** Protection exists for when it fails, not when it works

### Pattern 3: "It's Unnecessary Complexity"
> "This violates KISS, it should be simpler"

**Examples:**
- Multiple validation → "Consolidate into one function"
- Specific rate limiting → "Do it at infrastructure level"

**Reality:** Some protections require inherent complexity

### Pattern 4: "It's Premature Optimization"
> "This is optimization, we don't need that yet"

**Examples:**
- Resource limits → "We'll allocate resources when needed"
- Connection pooling → "Creating connections on demand is simpler"

**Reality:** Limits are protection, not optimization

---

## References

### Security by Language
- [Python Security](./python-security.md) - Pickle, eval, Django/Flask
- [TypeScript Security](./typescript-security.md) - eval, SQLi, Express
- [Go Security](./go-security.md) - Goroutines, SQL, FFI
- [Kotlin Security](./kotlin-security.md) - Null safety, Spring
- [Rust Security](./rust-security.md) - Unsafe, FFI, Ownership

### Performance by Language
- [Python Performance](../performance/python-performance.md) - GIL, asyncio, N+1
- [TypeScript Performance](../performance/typescript-performance.md) - Event loop, Workers
- [Go Performance](../performance/go-performance.md) - Goroutines, sync.Pool
- [Kotlin Performance](../performance/kotlin-performance.md) - Coroutines, Flow
- [Rust Performance](../performance/rust-performance.md) - Zero-cost, SIMD, Tokio
