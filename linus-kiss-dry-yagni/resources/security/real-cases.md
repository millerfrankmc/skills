# Casos Reales de Seguridad Eliminada por Error

> Este archivo documenta casos específicos donde la skill KISS-DRY-YAGNI eliminó código de seguridad por error. Úsalo como referencia para entender qué protecciones son fáciles de confundir con "código por si acaso".

---

## Índice de Protecciones

### 📚 Documentación por Lenguaje

| Lenguaje | Seguridad | Rendimiento |
|----------|-----------|-------------|
| **Python** | [python-security.md](./python-security.md) | [python-performance.md](../performance/python-performance.md) |
| **TypeScript** | [typescript-security.md](./typescript-security.md) | [typescript-performance.md](../performance/typescript-performance.md) |
| **Go** | [go-security.md](./go-security.md) | [go-performance.md](../performance/go-performance.md) |
| **Kotlin** | [kotlin-security.md](./kotlin-security.md) | [kotlin-performance.md](../performance/kotlin-performance.md) |
| **Rust** | [rust-security.md](./rust-security.md) | [rust-performance.md](../performance/rust-performance.md) |

### 🔒 Casos Documentados en Este Archivo

| Caso | Categoría | Descripción |
|------|-----------|-------------|
| [Caso 1: Shutdown Graceful](#caso-1-shutdown-graceful) | Protección de Procesos | Manejo de SIGTERM para cierre ordenado |
| [Caso 2: Usuario No-Root en Docker](#caso-2-usuario-no-root-en-docker) | Protección de Infraestructura | Seguridad en contenedores |
| [Caso 3: Circuit Breaker](#caso-3-circuit-breaker-eliminado) | Resiliencia | Prevención de cascada de fallos |
| [Caso 4: Rate Limiting](#caso-4-rate-limiting-eliminado) | Protección contra Abuso | Prevención de fuerza bruta |
| [Caso 5: Validación Múltiple](#caso-5-validación-múltiple-eliminada) | Defensa en Profundidad | Validación de uploads |
| [Caso 6: Resource Limits](#caso-6-resource-limits-eliminados) | Protección de Infraestructura | Límites en Kubernetes |

### ⚡ Optimizaciones de Rendimiento por Lenguaje

Consulta los archivos de performance específicos para cada lenguaje:

- **Python**: GIL y threading vs multiprocessing, generadores vs listas, asyncio, N+1 queries
- **TypeScript**: Event loop y async/await, Promise.all vs loops seriales, Worker threads
- **Go**: Goroutines y channels, sync.Pool, strings.Builder, pre-allocación de slices
- **Kotlin**: Coroutines vs threads, lazy initialization, inline functions, Flow
- **Rust**: Zero-cost abstractions, iterators, SIMD, async/await con Tokio

---

---

## Caso 1: Shutdown Graceful

**Fecha:** 2026-03-13
**Contexto:** Servicio Node.js con manejo de señales
**Archivo:** `src/server.ts`

### Código Eliminado

```javascript
process.on('SIGTERM', async () => {
  logger.info('Received SIGTERM, starting graceful shutdown...');

  // Dejar de aceptar nuevas conexiones
  server.close(async () => {
    logger.info('HTTP server closed');

    try {
      // Cerrar conexiones a base de datos
      await db.close();
      logger.info('Database connections closed');

      // Cerrar conexiones a cache
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

### Razón Incorrecta de Eliminación
> "Este código es 'por si acaso'. El sistema operativo ya maneja la terminación de procesos automáticamente. Además, el código es demasiado largo (20+ líneas) para una función que solo cierra conexiones."

### Impacto Real

1. **Corrupción de datos:** Requests en curso fueron interrumpidos a mitad de procesamiento
2. **Conexiones colgadas:** Conexiones a base de datos quedaron en estado `IDLE IN TRANSACTION`
3. **Pérdida de transacciones:** 3 transacciones financieras quedaron en estado inconsistente
4. **Reconexión lenta:** El siguiente despliegue tardó 5 minutos extra limpiando conexiones fantasmas

### Por Qué Es Seguridad (No YAGNI)

```
┌─────────────────────────────────────────────────────────────┐
│  Sin graceful shutdown          Con graceful shutdown       │
├─────────────────────────────────────────────────────────────┤
│  1. SIGTERM recibido            1. SIGTERM recibido         │
│  2. Proceso muere INMEDIATO     2. Señal capturada          │
│  3. Requests abortados          3. Dejar de aceptar nuevas  │
│  4. DB en estado inconsistente  4. Esperar requests activas │
│  5. Datos corruptos             5. Cerrar conexiones        │
│                                 6. Salir limpiamente        │
└─────────────────────────────────────────────────────────────┘
```

### Clasificación

- **Categoría:** Protección de Procesos
- **Subcategoría:** Shutdown Graceful
- **Palabras clave:** `SIGTERM`, `SIGINT`, `process.on`, `server.close()`, `cleanup`

---

## Caso 2: Usuario No-Root en Docker

**Fecha:** 2026-03-13
**Contexto:** Dockerfile de aplicación Python/Flask
**Archivo:** `Dockerfile`

### Código Eliminado

```dockerfile
FROM python:3.11-slim

# Instalar dependencias del sistema
RUN apt-get update && apt-get install -y \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Crear usuario no-root para ejecutar la aplicación
RUN groupadd -r appgroup && useradd -r -g appgroup appuser

# Crear directorio de trabajo
WORKDIR /app

# Copiar requirements primero (para caché de capas)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copiar código de la aplicación
COPY . .

# Cambiar propietario del código al usuario no-root
RUN chown -R appuser:appgroup /app

# Cambiar al usuario no-root
USER appuser

# Exponer puerto
EXPOSE 5000

# Comando de inicio
CMD ["python", "app.py"]
```

### Razón Incorrecta de Eliminación
> "La creación de usuario es complicación innecesaria. Los contenedores Docker ya están aislados del host. Según YAGNI, si no hay evidencia de que alguien escape de contenedores, no necesitamos esta protección. Además, el contenedor necesita root para instalar dependencias."

### Impacto Real

1. **Privilege escalation:** Si se explota una vulnerabilidad en la app (ej: RCE via deserialización), el atacante tiene **root dentro del contenedor**
2. **Escape de contenedor:** Con root + vulnerabilidad de kernel (muy común), el atacante obtiene **root en el host**
3. **Ataque a infraestructura:** El atacante puede:
   - Acceder a otros contenedores
   - Montar volúmenes del host
   - Modificar configuraciones del sistema
   - Instalar backdoors persistentes

### Demostración del Riesgo

```bash
# Sin USER directive (corriendo como root)
$ docker exec vulnerable_app whoami
root

# Con una vulnerabilidad RCE, el atacante puede:
root@container:~# mount /dev/sda1 /mnt/host
root@container:~# cat /mnt/host/etc/shadow
root@container:~# echo "backdoor" >> /mnt/host/root/.bashrc
```

### Por Qué Es Seguridad (No YAGNI)

```
┌────────────────────────────────────────────────────────────────┐
│           Defense in Depth                                     │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│   Capa 1: Aislamiento de contenedor (no garantizado)          │
│   ├─ Namespace isolation                                      │
│   ├─ Cgroups                                                  │
│   └─ Capabilities                                             │
│                                                                │
│   Capa 2: Usuario no-root (esta protección)                   │
│   ├─ Si el atacante escapa, no tiene root en host            │
│   └─ Limita el impacto de RCE dentro del contenedor          │
│                                                                │
│   Capa 3: Seccomp, AppArmor (protección adicional)            │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Clasificación

- **Categoría:** Protección de Infraestructura
- **Subcategoría:** Contenedor Seguro
- **Palabras clave:** `USER`, `useradd`, `adduser`, `runAsNonRoot`, `securityContext`

---

## Caso 3: Circuit Breaker Eliminado

**Fecha:** 2025-11-20
**Contexto:** Microservicio de pagos con llamadas a API bancaria
**Archivo:** `src/services/bankApi.ts`

### Código Eliminado

```typescript
class BankApiCircuitBreaker {
  private failureCount = 0;
  private lastFailureTime: number | null = null;
  private state: 'CLOSED' | 'OPEN' | 'HALF_OPEN' = 'CLOSED';

  private readonly FAILURE_THRESHOLD = 5;
  private readonly TIMEOUT = 60000; // 1 minuto

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

### Razón Incorrecta de Eliminación
> "Este código es demasiado complejo (40+ líneas) para un simple contador de errores. El error handling normal es suficiente. Además, si la API bancaria falla, queremos saberlo inmediatamente, no 'abrir un circuito'."

### Impacto Real

1. **Cascada de fallos:** Cuando la API bancaria tuvo 30s de latencia, TODOS los servicios que dependían de ella también se degradaron
2. **Timeout masivo:** Cada request esperó 30s antes de fallar, agotando el pool de conexiones
3. **Denial of Service:** El servicio dejó de responder a nuevas requests por 15 minutos
4. **Pérdida de revenue:** 200 transacciones de pago no procesadas

### Por Qué Es Seguridad/Resiliencia (No YAGNI)

```
┌────────────────────────────────────────────────────────────────┐
│  Sin Circuit Breaker            Con Circuit Breaker           │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  Request 1: timeout 30s         Request 1: timeout 30s        │
│  Request 2: timeout 30s         Request 2: timeout 30s        │
│  Request 3: timeout 30s         Request 3: timeout 30s        │
│  ...                            Request 4: timeout 30s        │
│  Request 100: timeout 30s       Request 5: timeout 30s        │
│                                 ────────────────────────      │
│  Total: 3000s de espera         CIRCUIT BREAKER ABIERTO       │
│  Pool de conexiones agotado     Request 6: Fallo INMEDIATO    │
│  Servicio no disponible         (sin esperar)                 │
│                                 Servicio sigue disponible     │
│                                 para otras operaciones        │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Clasificación

- **Categoría:** Resiliencia
- **Subcategoría:** Circuit Breaker
- **Palabras clave:** `CircuitBreaker`, `failureCount`, `threshold`, `OPEN`, `CLOSED`

---

## Caso 4: Rate Limiting Eliminado

**Fecha:** 2025-09-15
**Contexto:** Endpoint de login de aplicación web
**Archivo:** `src/routes/auth.ts`

### Código Eliminado

```typescript
import rateLimit from 'express-rate-limit';

const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutos
  max: 5, // máximo 5 intentos
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

// Aplicar solo al endpoint de login
app.post('/auth/login', loginLimiter, async (req, res) => {
  // ... lógica de login
});
```

### Razón Incorrecta de Eliminación
> "El rate limiting debería hacerse a nivel de infraestructura (Nginx, CloudFlare), no en código. Esto viola YAGNI porque ya tenemos protección en el edge. Además, el código es complejo (14 líneas) para algo que el proxy debería manejar."

### Impacto Real

1. **Ataque de fuerza bruta:** 100,000 intentos de login contra cuentas de administradores
2. **Compromiso de cuentas:** 3 cuentas de admin fueron comprometidas usando passwords comunes
3. **Acceso no autorizado:** El atacante accedió al panel de administración
4. **Exfiltración de datos:** Datos de 10,000 usuarios fueron exportados

### Por Qué Es Seguridad (No YAGNI)

```
┌────────────────────────────────────────────────────────────────┐
│  Defense in Depth para Autenticación                           │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  Capa 1: Edge (CloudFlare/Nginx)                              │
│  ├─ Protección DDoS                                           │
│  ├─ Rate limiting por IP                                      │
│  └─ Puede ser bypassed con rotación de IPs                   │
│                                                                │
│  Capa 2: Aplicación (este rate limiting)                      │
│  ├─ Rate limiting por cuenta (no solo IP)                    │
│  ├─ Protección contra credential stuffing                     │
│  └─ Logging de intentos sospechosos                           │
│                                                                │
│  Capa 3: Base de datos                                        │
│  ├─ Bloqueo de cuentas después de N intentos                  │
│  └─ Notificación al usuario                                   │
│                                                                │
│  Si eliminas Capa 2 → atacante puede atacar desde múltiples  │
│  IPs sin ser detectado                                        │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Clasificación

- **Categoría:** Protección contra Abuso
- **Subcategoría:** Rate Limiting
- **Palabras clave:** `rateLimit`, `max`, `windowMs`, `handler`

---

## Caso 5: Validación Múltiple Eliminada

**Fecha:** 2025-07-10
**Contexto:** Upload de archivos en aplicación de documentos
**Archivo:** `src/services/fileUpload.ts`

### Código Eliminado

```typescript
async function validateAndProcessFile(file: UploadedFile) {
  // Capa 1: Validar MIME type del header
  const allowedMimeTypes = ['application/pdf', 'image/jpeg', 'image/png'];
  if (!allowedMimeTypes.includes(file.mimetype)) {
    throw new ValidationError('Invalid MIME type');
  }

  // Capa 2: Validar extensión de archivo
  const ext = path.extname(file.originalname).toLowerCase();
  const allowedExtensions = ['.pdf', '.jpg', '.jpeg', '.png'];
  if (!allowedExtensions.includes(ext)) {
    throw new ValidationError('Invalid file extension');
  }

  // Capa 3: Validar magic bytes (contenido real del archivo)
  const magicBytes = await readFirstBytes(file.path, 8);
  if (!isValidMagicBytes(magicBytes, file.mimetype)) {
    throw new ValidationError('File content does not match declared type');
  }

  // Capa 4: Validar tamaño máximo
  const MAX_SIZE = 10 * 1024 * 1024; // 10MB
  if (file.size > MAX_SIZE) {
    throw new ValidationError('File too large');
  }

  // Capa 5: Scan con antivirus (ClamAV)
  const scanResult = await clamav.scan(file.path);
  if (scanResult.isInfected) {
    throw new ValidationError('File contains malware');
  }

  return processFile(file);
}
```

### Razón Incorrecta de Eliminación
> "Esta función viola DRY - tiene 5 validaciones diferentes para el 'mismo' archivo. Deberíamos consolidar en una sola función `validateFile()`. Además, validar MIME type Y extensión Y magic bytes es redundante - si el MIME type es correcto, el archivo es válido."

### Impacto Real

1. **Bypass de seguridad:** Un atacante subió un archivo `.exe` renombrado a `.pdf`
2. **Ejecución de malware:** El archivo explotó una vulnerabilidad en el visor de PDF
3. **Compromiso del servidor:** Se instaló un cryptominer
4. **Lateral movement:** El atacante accedió a la red interna

### Por Qué Es Seguridad (No DRY)

```
┌────────────────────────────────────────────────────────────────┐
│  Defensa en Profundidad - Cada capa protege contra diferente  │
│  vector de ataque                                             │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  Capa 1: MIME type                                             │
│  └─ Protege contra: Browser ejecutando archivo como script    │
│                                                                │
│  Capa 2: Extensión de archivo                                  │
│  └─ Protege contra: Confusión de tipo en el sistema de archivos│
│                                                                │
│  Capa 3: Magic bytes                                           │
│  └─ Protege contra: MIME spoofing (header falso)               │
│                                                                │
│  Capa 4: Tamaño                                                │
│  └─ Protege contra: DoS por disco lleno                        │
│                                                                │
│  Capa 5: Antivirus                                             │
│  └─ Protege contra: Malware conocido                           │
│                                                                │
│  Si consolidas en una sola validación → bypass fácil           │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Clasificación

- **Categoría:** Defensa en Profundidad
- **Subcategoría:** Validación Múltiple
- **Palabras clave:** `mimetype`, `extension`, `magic bytes`, `validation`

---

## Caso 6: Resource Limits Eliminados

**Fecha:** 2025-05-22
**Contexto:** Deployment en Kubernetes
**Archivo:** `k8s/deployment.yaml`

### Código Eliminado

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

### Razón Incorrecta de Eliminación
> "Los resource limits son una optimización prematura. Kubernetes asignará recursos dinámicamente según sea necesario. Además, estos valores son arbitrarios (¿por qué 512Mi y no 1Gi?). Cuando necesitemos más recursos, los ajustaremos."

### Impacto Real

1. **Memory leak en producción:** Un bug causó que el consumo de memoria creciera indefinidamente
2. **OOM Kill en cascada:** Sin límites, el pod consumió 8GB de RAM y fue killed
3. **Node pressure:** El nodo completo quedó sin memoria
4. **Evicción de pods:** Kubernetes empezó a matar otros pods para liberar memoria
5. **Degradación total:** Toda la aplicación quedó inusable por 30 minutos

### Por Qué Es Seguridad/Estabilidad (No Optimización Prematura)

```
┌────────────────────────────────────────────────────────────────┐
│  Sin Resource Limits            Con Resource Limits           │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  Memory leak ocurre             Memory leak ocurre            │
│  ├─ Consumo crece sin límite    ├─ Alcanza límite (512Mi)     │
│  ├─ Pod usa 8GB RAM             ├─ Pod es OOM Killed          │
│  ├─ Nodo sin memoria            ├─ Kubernetes reinicia pod    │
│  ├─ Otros pods eviccionados     ├─ Sistema sigue estable      │
│  └─ Degradación total           └─ Un solo pod afectado       │
│                                                                │
│  Límite = Fail fast + Contención de daños                     │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Clasificación

- **Categoría:** Protección de Infraestructura
- **Subcategoría:** Resource Limits
- **Palabras clave:** `resources`, `limits`, `requests`, `memory`, `cpu`

---

## Patrones Comunes de Error

### Patrón 1: "El Sistema Ya Lo Maneja"
> "El OS/Container/Kubernetes ya maneja eso"

**Ejemplos:**
- Shutdown graceful → "El OS mata el proceso"
- Usuario no-root → "Docker ya aísla"
- Rate limiting → "Nginx ya lo hace"

**Realidad:** Defense in depth - cada capa es necesaria

### Patrón 2: "Nunca Ha Fallado"
> "Llevamos 2 años sin que esto sea un problema"

**Ejemplos:**
- Circuit breaker → "Nunca hemos tenido cascada de fallos"
- Resource limits → "Nunca hemos tenido memory leaks"

**Realidad:** La protección existe para cuando falle, no para cuando funciona

### Patrón 3: "Es Complejidad Innecesaria"
> "Esto viola KISS, debería ser más simple"

**Ejemplos:**
- Validación múltiple → "Consolidar en una función"
- Rate limiting específico → "Hacerlo a nivel de infraestructura"

**Realidad:** Algunas protecciones requieren complejidad inherente

### Patrón 4: "Es Optimización Prematura"
> "Esto es optimización, no necesitamos eso todavía"

**Ejemplos:**
- Resource limits → "Asignaremos recursos cuando sea necesario"
- Connection pooling → "Crear conexiones bajo demanda es más simple"

**Realidad:** Límites son protección, no optimización

---

## Referencias

### Seguridad por Lenguaje
- [Python Security](./python-security.md) - Pickle, eval, Django/Flask
- [TypeScript Security](./typescript-security.md) - eval, SQLi, Express
- [Go Security](./go-security.md) - Goroutines, SQL, FFI
- [Kotlin Security](./kotlin-security.md) - Null safety, Spring
- [Rust Security](./rust-security.md) - Unsafe, FFI, Ownership

### Rendimiento por Lenguaje
- [Python Performance](../performance/python-performance.md) - GIL, asyncio, N+1
- [TypeScript Performance](../performance/typescript-performance.md) - Event loop, Workers
- [Go Performance](../performance/go-performance.md) - Goroutines, sync.Pool
- [Kotlin Performance](../performance/kotlin-performance.md) - Coroutines, Flow
- [Rust Performance](../performance/rust-performance.md) - Zero-cost, SIMD, Tokio
