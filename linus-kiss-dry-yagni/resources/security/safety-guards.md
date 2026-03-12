# Mecanismos de Protección de Seguridad

## Visión General

Esta skill aplica KISS-DRY-YAGNI de forma automática, pero con una excepción crítica: **el código de seguridad nunca se simplifica a expensas de la protección**. Estos mecanismos detectan código de seguridad y aplican reglas diferenciadas.

---

## Mecanismo 1: Detector de Validaciones Críticas

### Funcionamiento

Análisis estático que identifica código de seguridad mediante patrones heurísticos en tres categorías.

### Patrones Detectados

#### 1.1 Nombres de Funciones/Variables
```typescript
// Prefijos que indican seguridad
validate*     → validateInput, validateToken, validateSession
sanitize*     → sanitizeInput, sanitizeHTML, sanitizeSQL
check*        → checkPermission, checkRateLimit, checkCSRF
verify*       → verifySignature, verifyJWT, verifyHash
authenticate* → authenticateUser, authenticateRequest
authorize*    → authorizeAccess, authorizeAction
hash*         → hashPassword, hashData
encrypt*      → encryptData, encryptField
decrypt*      → decryptPayload, decryptToken
escape*       → escapeHTML, escapeSQL, escapeRegex
scrub*        → scrubPII, scrubSensitiveData
isValid*      → isValidToken, isValidSession
hasPermission → hasPermission, hasRole, hasAccess
secure*       → secureCompare, secureRandom
rateLimit*    → rateLimitRequest, rateLimitLogin
csrf*         → csrfToken, csrfProtection
xss*          → xssFilter, xssSanitize
```

#### 1.2 Bibliotecas Criptográficas
```typescript
// Importaciones que activan modo seguro
import * as crypto from 'crypto';           // Node.js crypto
import bcrypt from 'bcrypt';               // Password hashing
import argon2 from 'argon2';               // Modern password hashing
import jwt from 'jsonwebtoken';            // JWT handling
import { createHmac } from 'crypto';        // HMAC operations
import helmet from 'helmet';               // Security headers
import csrf from 'csurf';                  // CSRF protection
import rateLimit from 'express-rate-limit'; // Rate limiting
import DOMPurify from 'dompurify';         // XSS sanitization
import validator from 'validator';         // Input validation
import owasp from 'owasp-password-strength-test'; // Password policy
```

#### 1.3 Expresiones Regulares de Validación
```typescript
// Regex patterns que indican validación de seguridad
const SECURITY_PATTERNS = [
  /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/,       // Password complexity
  /^[a-f0-9]{64}$/i,                       // SHA-256 hash
  /^[a-f0-9]{128}$/i,                      // SHA-512 hash
  /^\$2[aby]\$\d+\$/,                      // bcrypt hash
  /^[A-Za-z0-9-_]+={0,2}$/,                // Base64
  /^Bearer\s+[A-Za-z0-9-_\.]+/,            // Bearer token
  /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}/,  // UUID
  /^(https?):\/\/[^\s/$.?#].[^\s]*$/i,    // URL validation
  /^[^\s@]+@[^\s@]+\.[^\s@]+$/,           // Email (when used for auth)
];
```

#### 1.4 Comparaciones de Seguridad
```typescript
// Operaciones que deben usar comparación segura
// Detecta: comparaciones de contraseñas, tokens, secrets
if (input === storedPassword)           // INSEGURO - detectar
if (token === expectedToken)            // INSEGURO - detectar
crypto.timingSafeEqual(a, b)            // SEGURO - preservar

// Headers de seguridad
'X-CSRF-Token'
'X-Frame-Options'
'Content-Security-Policy'
'X-Content-Type-Options'
'Strict-Transport-Security'
'X-XSS-Protection'
```

### Integración en Flujo

```typescript
// Pseudocódigo del detector
function analyzeFunction(func: Function): SecurityClassification {
  const indicators = {
    securityPrefixes: countSecurityPrefixes(func.name),
    cryptoImports: detectCryptoLibraries(func.imports),
    securityRegex: detectSecurityPatterns(func.code),
    sensitiveComparisons: detectSensitiveComparisons(func.code),
    securityHeaders: detectSecurityHeaders(func.code),
  };

  const score = calculateSecurityScore(indicators);

  if (score >= SECURITY_THRESHOLD) {
    return {
      type: 'SECURITY_CRITICAL',
      rules: SECURITY_RULES,  // Reglas diferenciadas
      allowExceedLimits: true,
      preventRemoval: true,
    };
  }

  return { type: 'NORMAL', rules: STANDARD_RULES };
}
```

---

## Mecanismo 2: Reglas Diferenciadas para Código de Seguridad

### Tabla de Reglas por Tipo

| Aspecto | Código Normal | Código de Seguridad |
|---------|---------------|---------------------|
| **Líneas por función** | Max 20 | Max 50 (con justificación) |
| **Niveles de anidamiento** | Max 2 | Max 4 (para validaciones) |
| **Parámetros** | Max 4 | Max 6 (opciones de seguridad) |
| **Comentarios** | Eliminar "qué" | Preservar "por qué" de seguridad |
| **Interfaces** | Eliminar si 1 uso | Mantener (extensibilidad) |
| **Código "por si acaso"** | Eliminar | Preservar (defensa en profundidad) |
| **Guard clauses** | Aplicar | Aplicar con cuidado (no perder validaciones) |

### Ejemplo: Validación de Input

```typescript
// ANTES: Función de seguridad de 35 líneas
// (El detector identifica: validate*, regex de seguridad, sanitización)

function validateAndSanitizeUserInput(input: unknown): SanitizedInput {
  // SECURITY: Esta función es crítica - no simplificar a expensas de seguridad

  if (!input || typeof input !== 'object') {
    throw new ValidationError('Input must be an object');
  }

  const result: SanitizedInput = {};

  // Validar nombre - permite letras, espacios, apóstrofes
  if ('name' in input) {
    const name = String(input.name).trim();
    if (name.length < 1 || name.length > 100) {
      throw new ValidationError('Name must be 1-100 characters');
    }
    if (!/^[\p{L}\s'-]+$/u.test(name)) {
      throw new ValidationError('Name contains invalid characters');
    }
    result.name = escapeHtml(name);
  }

  // Validar email con regex estricta
  if ('email' in input) {
    const email = String(input.email).toLowerCase().trim();
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
      throw new ValidationError('Invalid email format');
    }
    if (email.length > 254) {
      throw new ValidationError('Email too long');
    }
    result.email = email;
  }

  // Validar edad - rango seguro
  if ('age' in input) {
    const age = Number(input.age);
    if (!Number.isInteger(age) || age < 0 || age > 150) {
      throw new ValidationError('Age must be an integer 0-150');
    }
    result.age = age;
  }

  // Verificar campos no permitidos (prevenir injection)
  const allowedFields = ['name', 'email', 'age'];
  const extraFields = Object.keys(input).filter(k => !allowedFields.includes(k));
  if (extraFields.length > 0) {
    throw new ValidationError(`Unexpected fields: ${extraFields.join(', ')}`);
  }

  return result;
}

// DESPUÉS: Aplicando KISS manteniendo seguridad
// Se permite >20 líneas porque es código de seguridad
// Se mantiene cada validación - ninguna se elimina como "por si acaso"

// Lo que NO se hace (aunque reduce líneas):
// ❌ Eliminar validación de longitud máxima
// ❌ Eliminar validación de caracteres permitidos
// ❌ Eliminar verificación de campos extra
// ❌ Usar validación genérica que pierde especificidad
```

### Ejemplo: Rate Limiting

```typescript
// ANTES: Código de rate limiting con 30 líneas
// (Detector identifica: rateLimit*, headers de seguridad, prevención de ataque)

function rateLimitRequest(
  identifier: string,
  endpoint: string,
  config: RateLimitConfig
): RateLimitResult {
  const key = `ratelimit:${endpoint}:${identifier}`;
  const now = Date.now();
  const windowStart = now - config.windowMs;

  // Limpiar entradas antiguas
  const requests = getRequestsFromStore(key);
  const validRequests = requests.filter(ts => ts > windowStart);

  // Verificar límite
  if (validRequests.length >= config.maxRequests) {
    const oldestRequest = Math.min(...validRequests);
    const retryAfter = Math.ceil((oldestRequest + config.windowMs - now) / 1000);

    return {
      allowed: false,
      retryAfter,
      remaining: 0,
      headers: {
        'X-RateLimit-Limit': String(config.maxRequests),
        'X-RateLimit-Remaining': '0',
        'X-RateLimit-Reset': String(Math.ceil((oldestRequest + config.windowMs) / 1000)),
        'Retry-After': String(retryAfter),
      },
    };
  }

  // Registrar request actual
  validRequests.push(now);
  storeRequests(key, validRequests, config.windowMs);

  return {
    allowed: true,
    remaining: config.maxRequests - validRequests.length,
    headers: {
      'X-RateLimit-Limit': String(config.maxRequests),
      'X-RateLimit-Remaining': String(config.maxRequests - validRequests.length),
    },
  };
}

// DESPUÉS: Se mantiene completo
// Reglas aplicadas:
// ✅ Permitir >20 líneas (es rate limiting)
// ✅ Mantener todos los headers de seguridad
// ✅ Preservar lógica de ventana deslizante
// ✅ No simplificar cálculo de retry-after
```

---

## Mecanismo 3: Lista Blanca de Seguridad (Never-Touch List)

### Elementos Protegidos Absolutos

Estos elementos NUNCA se modifican, simplifican o eliminan:

#### 3.1 Password Hashing
```typescript
// NUNCA simplificar o modificar
const hash = await bcrypt.hash(password, 12);  // Cost factor se mantiene
const isValid = await bcrypt.compare(password, hash);  // Comparación segura interna

// NUNCA reemplazar con comparación simple
// ❌ if (hashPassword(input) === storedHash)  // INSEGURO - no hacer
// ✅ await bcrypt.compare(input, storedHash)   // Mantener
```

#### 3.2 SQL Parameterization
```typescript
// NUNCA eliminar parameterization
// ❌ `SELECT * FROM users WHERE id = ${userId}`  // INYECCIÓN - nunca
// ✅ db.query('SELECT * FROM users WHERE id = ?', [userId])  // Mantener

// NUNCA simplificar a concatenación
// ❌ query += ' AND name = \'' + name + '\''  // NUNCA
```

#### 3.3 CSRF Protection
```typescript
// NUNCA eliminar validación CSRF
app.use(csrf({ cookie: true }));  // Mantener

// NUNCA simplificar verificación de token
// ❌ if (token) { proceed() }  // Insuficiente
// ✅ if (!req.csrfToken || req.csrfToken() !== req.body._csrf) {
//      throw new CSRFError();
//    }
```

#### 3.4 CORS Configuration
```typescript
// NUNCA simplificar a "permitir todo"
// ❌ app.use(cors({ origin: '*' }))  // INSEGURO para APIs con auth

// ✅ Mantener configuración explícita
app.use(cors({
  origin: ['https://app.example.com', 'https://admin.example.com'],
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: true,
  maxAge: 86400,
}));
```

#### 3.5 Input Validation
```typescript
// NUNCA eliminar validaciones "redundantes"
// ❌ "Ya valida en el frontend" - NO es excusa
// ❌ "El ORM lo maneja" - NO es excusa

// ✅ Mantener validación en múltiples capas
function createUser(data: unknown) {
  // Validación 1: Tipo
  if (!data || typeof data !== 'object') throw new Error();

  // Validación 2: Campos requeridos
  if (!('email' in data)) throw new Error();

  // Validación 3: Formato
  if (!isValidEmail(data.email)) throw new Error();

  // Validación 4: Longitud
  if (data.email.length > 254) throw new Error();

  // ... cada capa tiene propósito
}
```

#### 3.6 Secure Comparisons
```typescript
// NUNCA reemplazar timingSafeEqual
const isValid = crypto.timingSafeEqual(
  Buffer.from(provided),
  Buffer.from(expected)
);

// NUNCA usar comparación normal para secrets
// ❌ if (provided === expected)  // Vulnerable a timing attacks
```

#### 3.7 Security Headers
```typescript
// NUNCA eliminar headers de seguridad
// Todos estos se preservan:
{
  'Strict-Transport-Security': 'max-age=31536000; includeSubDomains',
  'Content-Security-Policy': "default-src 'self'",
  'X-Content-Type-Options': 'nosniff',
  'X-Frame-Options': 'DENY',
  'X-XSS-Protection': '1; mode=block',
  'Referrer-Policy': 'strict-origin-when-cross-origin',
}
```

### Implementación de Lista Blanca

```typescript
const NEVER_TOUCH_PATTERNS = [
  // Password hashing
  { pattern: /bcrypt\.(hash|compare)/, reason: 'Password hashing' },
  { pattern: /argon2\.(hash|verify)/, reason: 'Password hashing' },
  { pattern: /crypto\.timingSafeEqual/, reason: 'Timing-safe comparison' },

  // SQL injection prevention
  { pattern: /\.query\s*\([^,]+,\s*\[/, reason: 'SQL parameterization' },
  { pattern: /prepare\s*\([^)]+\?/, reason: 'Prepared statements' },

  // CSRF
  { pattern: /csrfToken\s*\(\)/, reason: 'CSRF token generation' },
  { pattern: /req\.csrfToken/, reason: 'CSRF validation' },

  // CORS
  { pattern: /cors\s*\(\s*\{[^}]*origin:/, reason: 'CORS configuration' },

  // Security headers
  { pattern: /Strict-Transport-Security/, reason: 'HSTS header' },
  { pattern: /Content-Security-Policy/, reason: 'CSP header' },
  { pattern: /X-Frame-Options/, reason: 'Clickjacking protection' },

  // Input validation libraries
  { pattern: /validator\.(isEmail|isURL|isUUID)/, reason: 'Input validation' },
  { pattern: /DOMPurify\.sanitize/, reason: 'XSS prevention' },
];

function isNeverTouch(code: string): boolean {
  return NEVER_TOUCH_PATTERNS.some(({ pattern }) => pattern.test(code));
}
```

---

## Mecanismo 4: Marcadores de Código Crítico

### Comentarios Especiales de Seguridad

Los desarrolladores pueden usar comentarios especiales que la skill reconoce y respeta:

```typescript
// SECURITY: [explicación] - Marca código como crítico
// SECURITY-CRITICAL: [explicación] - No tocar bajo ninguna circunstancia
// SECURITY-NO-SIMPLIFY: [razón] - No aplicar simplificación
// SECURITY-PRESERVE: [qué] - Preservar comportamiento específico

// Ejemplos de uso:

// SECURITY: Validación de entrada crítica para prevenir XSS
function sanitizeHtml(input: string): string {
  // SECURITY-NO-SIMPLIFY: DOMPurify tiene configuración específica
  return DOMPurify.sanitize(input, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong'],
    ALLOWED_ATTR: [],
  });
}

// SECURITY-CRITICAL: Comparación de contraseñas
async function verifyPassword(input: string, hash: string): Promise<boolean> {
  // SECURITY-PRESERVE: Usar bcrypt.compare, nunca comparación directa
  return await bcrypt.compare(input, hash);
}

// SECURITY: Rate limiting para prevenir brute force
// SECURITY-NO-SIMPLIFY: La ventana de 15 minutos es requerimiento de seguridad
function checkLoginRateLimit(ip: string): boolean {
  // ... lógica de rate limiting ...
}
```

### Anotaciones/Estructuras de Nombres

Convenciones de nomenclatura que activan protección automática:

```typescript
// Prefijos que activan modo seguro
function securityValidate*()     // securityValidateInput, securityValidateToken
function auth*()                 // authLogin, authVerify
function permission*()           // permissionCheck, permissionRequire

// Sufijos que activan modo seguro
function *Secure()               // compareSecure, randomSecure
function *Validator()            // inputValidator, emailValidator
function *Guard()                // authGuard, csrfGuard

// Variables
const SECURITY_*                 // SECURITY_HEADERS, SECURITY_CONFIG
const CSRF_*                     // CSRF_TOKEN, CSRF_SECRET
const RATE_LIMIT_*               // RATE_LIMIT_MAX, RATE_LIMIT_WINDOW
```

### Integración con Marcadores

```typescript
function detectSecurityMarkers(code: string): SecurityMarkers {
  const markers = {
    comments: [] as string[],
    prefixes: [] as string[],
    suffixes: [] as string[],
  };

  // Detectar comentarios de seguridad
  const securityCommentRegex = /\/\/\s*(SECURITY[^\n]*)/gi;
  let match;
  while ((match = securityCommentRegex.exec(code)) !== null) {
    markers.comments.push(match[1]);
  }

  // Detectar prefijos de seguridad
  const prefixRegex = /\b(security|auth|permission)[A-Z]\w+/g;
  while ((match = prefixRegex.exec(code)) !== null) {
    markers.prefixes.push(match[0]);
  }

  // Detectar sufijos de seguridad
  const suffixRegex = /\b\w+(Secure|Validator|Guard)\b/g;
  while ((match = suffixRegex.exec(code)) !== null) {
    markers.suffixes.push(match[0]);
  }

  return markers;
}

function applyMarkerRules(markers: SecurityMarkers): RuleSet {
  if (markers.comments.some(m => m.includes('SECURITY-CRITICAL'))) {
    return NO_CHANGES_ALLOWED;  // No tocar nada
  }

  if (markers.comments.some(m => m.includes('SECURITY-NO-SIMPLIFY'))) {
    return PRESERVE_STRUCTURE;  // No simplificar estructura
  }

  if (markers.prefixes.length > 0 || markers.suffixes.length > 0) {
    return SECURITY_RULES;  // Reglas de seguridad
  }

  return STANDARD_RULES;
}
```

---

## Mecanismo 5: Verificación Post-Simplificación

### Validaciones Automáticas

Después de cualquier modificación, se ejecutan verificaciones:

```typescript
interface PostSimplificationCheck {
  name: string;
  validate: (before: Code, after: Code) => ValidationResult;
}

const POST_SIMPLIFICATION_CHECKS: PostSimplificationCheck[] = [
  {
    name: 'Validations Preserved',
    validate: (before, after) => {
      const beforeValidations = countValidationChecks(before);
      const afterValidations = countValidationChecks(after);

      if (afterValidations < beforeValidations) {
        return {
          passed: false,
          error: `Se eliminaron ${beforeValidations - afterValidations} validaciones`,
        };
      }
      return { passed: true };
    },
  },

  {
    name: 'Security Imports Preserved',
    validate: (before, after) => {
      const beforeImports = extractSecurityImports(before);
      const afterImports = extractSecurityImports(after);

      const missing = beforeImports.filter(i => !afterImports.includes(i));
      if (missing.length > 0) {
        return {
          passed: false,
          error: `Imports de seguridad eliminados: ${missing.join(', ')}`,
        };
      }
      return { passed: true };
    },
  },

  {
    name: 'Security Headers Preserved',
    validate: (before, after) => {
      const beforeHeaders = extractSecurityHeaders(before);
      const afterHeaders = extractSecurityHeaders(after);

      const missing = beforeHeaders.filter(h => !afterHeaders.includes(h));
      if (missing.length > 0) {
        return {
          passed: false,
          error: `Headers de seguridad eliminados: ${missing.join(', ')}`,
        };
      }
      return { passed: true };
    },
  },

  {
    name: 'Error Handling Preserved',
    validate: (before, after) => {
      const beforeErrors = countErrorHandling(before);
      const afterErrors = countErrorHandling(after);

      if (afterErrors < beforeErrors) {
        return {
          passed: false,
          error: `Manejo de errores reducido: ${beforeErrors} -> ${afterErrors}`,
        };
      }
      return { passed: true };
    },
  },

  {
    name: 'No Clever Code in Security',
    validate: (before, after) => {
      // Verificar que no se introdujo código "clever" en funciones de seguridad
      const securityFuncs = extractSecurityFunctions(after);

      for (const func of securityFuncs) {
        if (containsCleverCode(func)) {
          return {
            passed: false,
            error: `Código "clever" detectado en función de seguridad: ${func.name}`,
          };
        }
      }
      return { passed: true };
    },
  },
];
```

### Comparación de Comportamiento

```typescript
function verifyBehavioralEquivalence(before: Function, after: Function): boolean {
  // Casos de prueba de seguridad
  const testCases = [
    { input: null, expected: 'throw' },
    { input: undefined, expected: 'throw' },
    { input: '', expected: 'throw' },
    { input: 'a'.repeat(10000), expected: 'throw' },  // Longitud excesiva
    { input: '<script>alert(1)</script>', expected: 'sanitize' },
    { input: "'; DROP TABLE users; --", expected: 'sanitize' },
    { input: '../../../etc/passwd', expected: 'sanitize' },
  ];

  for (const testCase of testCases) {
    const beforeResult = safeExecute(() => before(testCase.input));
    const afterResult = safeExecute(() => after(testCase.input));

    if (!equivalentSecurityBehavior(beforeResult, afterResult, testCase.expected)) {
      return false;
    }
  }

  return true;
}
```

### Cobertura de Casos Edge

```typescript
const EDGE_CASES = {
  validation: [
    'null input',
    'undefined input',
    'empty string',
    'whitespace only',
    'maximum length + 1',
    'unicode characters',
    'special characters',
    'injection attempts',
  ],

  authentication: [
    'empty credentials',
    'null password',
    'password with null bytes',
    'very long password',
    'timing attack resistance',
  ],

  authorization: [
    'missing token',
    'expired token',
    'malformed token',
    'token with wrong signature',
    'token with elevated privileges',
  ],
};

function verifyEdgeCaseCoverage(before: Code, after: Code): CoverageResult {
  const beforeCoverage = analyzeEdgeCaseHandling(before, EDGE_CASES);
  const afterCoverage = analyzeEdgeCaseHandling(after, EDGE_CASES);

  const lostCoverage = beforeCoverage.filter(c => !afterCoverage.includes(c));

  if (lostCoverage.length > 0) {
    return {
      passed: false,
      lostCases: lostCoverage,
    };
  }

  return { passed: true };
}
```

---

## Mecanismo 6: Análisis de Flujo de Datos Sensibles

### Rastreo de Datos Sensibles

```typescript
const SENSITIVE_DATA_TYPES = [
  'password',
  'token',
  'secret',
  'key',
  'credential',
  'ssn',
  'creditCard',
  'personalData',
];

function trackSensitiveDataFlow(code: Code): DataFlowGraph {
  const graph: DataFlowGraph = { nodes: [], edges: [] };

  // Identificar fuentes de datos sensibles
  const sources = findVariableDeclarations(code, v =>
    SENSITIVE_DATA_TYPES.some(type =>
      v.name.toLowerCase().includes(type)
    )
  );

  // Rastrear flujo a través del código
  for (const source of sources) {
    const flows = traceDataFlow(code, source);
    graph.nodes.push(...flows.nodes);
    graph.edges.push(...flows.edges);
  }

  return graph;
}

function verifySecureHandling(graph: DataFlowGraph): SecurityReport {
  const issues: SecurityIssue[] = [];

  for (const edge of graph.edges) {
    // Verificar que datos sensibles no se loggean
    if (edge.target.type === 'log' && !edge.isSanitized) {
      issues.push({
        severity: 'CRITICAL',
        message: `Dato sensible '${edge.source.name}' podría loggearse`,
      });
    }

    // Verificar que no se almacenan en texto plano
    if (edge.target.type === 'storage' && !edge.isEncrypted) {
      issues.push({
        severity: 'CRITICAL',
        message: `Dato sensible '${edge.source.name}' almacenado sin encriptar`,
      });
    }

    // Verificar que no se envían por HTTP
    if (edge.target.type === 'http' && !edge.isOverHttps) {
      issues.push({
        severity: 'HIGH',
        message: `Dato sensible '${edge.source.name}' enviado por HTTP`,
      });
    }
  }

  return { issues };
}
```

---

## Mecanismo 7: Contexto de Seguridad por Dominio

### Reglas por Tipo de Archivo

```typescript
const DOMAIN_SECURITY_RULES: Record<string, SecurityRules> = {
  'auth': {
    // Archivos de autenticación
    maxLines: 50,
    preserveInterfaces: true,
    requireComments: true,
    neverRemove: ['password', 'hash', 'salt', 'token'],
  },

  'validation': {
    // Archivos de validación
    maxLines: 40,
    preserveAllChecks: true,
    requireExplicitValidation: true,
  },

  'middleware/security': {
    // Middleware de seguridad
    maxLines: 60,
    preserveHeaders: true,
    preserveOrder: true,  // Orden de middleware importa
  },

  'crypto': {
    // Operaciones criptográficas
    maxLines: 30,
    noChanges: true,  // Casi no tocar
    requireExpertReview: true,
  },

  'config/security': {
    // Configuración de seguridad
    maxLines: 100,
    preserveAll: true,
    requireComments: true,
  },
};

function detectDomain(filePath: string): string {
  if (filePath.includes('/auth/')) return 'auth';
  if (filePath.includes('/validation/')) return 'validation';
  if (filePath.includes('/middleware/') && filePath.includes('security')) return 'middleware/security';
  if (filePath.includes('/crypto/')) return 'crypto';
  if (filePath.includes('/config/') && filePath.includes('security')) return 'config/security';
  return 'default';
}
```

---

## Flujo de Trabajo Integrado

```
┌─────────────────────────────────────────────────────────────────┐
│                    FLUJO KISS-DRY-YAGNI + SEGURIDAD             │
└─────────────────────────────────────────────────────────────────┘

1. ANÁLISIS INICIAL
   ├─ Detectar imports de seguridad
   ├─ Detectar nombres de funciones/variables de seguridad
   ├─ Detectar patrones de regex de seguridad
   ├─ Detectar comentarios SECURITY:
   ├─ Detectar dominio del archivo
   └─ Calcular score de seguridad

2. CLASIFICACIÓN
   ├─ Score >= THRESHOLD → Modo SEGURIDAD
   │   ├─ Aplicar reglas diferenciadas
   │   ├─ Verificar lista blanca
   │   └─ Preservar marcadores
   └─ Score < THRESHOLD → Modo ESTÁNDAR
       └─ Aplicar KISS-DRY-YAGNI normal

3. TRANSFORMACIÓN (Modo Seguridad)
   ├─ Permitir >20 líneas con justificación
   ├─ NO eliminar validaciones
   ├─ NO eliminar imports de seguridad
   ├─ NO simplificar headers de seguridad
   ├─ NO eliminar interfaces de seguridad
   ├─ Preservar comentarios SECURITY:
   └─ Aplicar guard clauses con cuidado

4. VERIFICACIÓN POST-SIMPLIFICACIÓN
   ├─ ¿Se preservaron todas las validaciones?
   ├─ ¿Se preservaron imports de seguridad?
   ├─ ¿Se preservaron headers de seguridad?
   ├─ ¿Se preservó manejo de errores?
   ├─ ¿No se introdujo código clever?
   ├─ ¿Se mantiene cobertura de casos edge?
   └─ ¿Flujo de datos sensibles es seguro?

5. DECISIÓN FINAL
   ├─ Todas las verificaciones pasan → Aceptar cambios
   └─ Alguna verificación falla →
       ├─ Revertir cambios problemáticos
       ├─ Aplicar solo cambios seguros
       └─ Reportar qué se preservó y por qué

6. REPORTE
   ├─ [KISS] Cambios aplicados (si aplica)
   ├─ [DRY] Cambios aplicados (si aplica)
   ├─ [YAGNI] Cambios aplicados (si aplica)
   ├─ [SECURITY] Qué se preservó y por qué
   └─ [SECURITY] Verificaciones post-simplificación
```

---

## Ejemplo Completo: Refactorización con Protección

```typescript
// ============================================================
// ARCHIVO: src/auth/login.ts
// ============================================================

// ANTES: Código con problemas de complejidad Y elementos de seguridad

import bcrypt from 'bcrypt';
import jwt from 'jsonwebtoken';
import rateLimit from 'express-rate-limit';

// SECURITY: Configuración de rate limiting para prevenir brute force
const loginRateLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutos
  max: 5,  // 5 intentos
  message: 'Too many login attempts',
  standardHeaders: true,
  legacyHeaders: false,
});

// Función God con 45 líneas - mezcla seguridad y lógica de negocio
async function handleLogin(req: Request, res: Response) {
  // Validación básica
  if (!req.body || !req.body.email || !req.body.password) {
    return res.status(400).json({ error: 'Missing credentials' });
  }

  const { email, password } = req.body;

  // Buscar usuario
  const user = await db.users.findOne({ where: { email } });
  if (!user) {
    // SECURITY: Tiempo constante para prevenir timing attacks
    await bcrypt.compare('dummy', '$2b$10$dummyhashfordummyuser');
    return res.status(401).json({ error: 'Invalid credentials' });
  }

  // Verificar si cuenta está bloqueada
  if (user.lockedUntil && user.lockedUntil > new Date()) {
    return res.status(423).json({
      error: 'Account locked',
      retryAfter: Math.ceil((user.lockedUntil - Date.now()) / 1000),
    });
  }

  // Verificar contraseña
  const isValidPassword = await bcrypt.compare(password, user.passwordHash);
  if (!isValidPassword) {
    // Incrementar contador de fallos
    user.failedLoginAttempts = (user.failedLoginAttempts || 0) + 1;

    if (user.failedLoginAttempts >= 5) {
      user.lockedUntil = new Date(Date.now() + 15 * 60 * 1000);
      await user.save();
      return res.status(423).json({ error: 'Account locked for 15 minutes' });
    }

    await user.save();
    return res.status(401).json({ error: 'Invalid credentials' });
  }

  // Resetear contador de fallos
  user.failedLoginAttempts = 0;
  user.lockedUntil = null;
  user.lastLoginAt = new Date();
  await user.save();

  // Generar JWT
  const token = jwt.sign(
    { userId: user.id, email: user.email },
    process.env.JWT_SECRET!,
    { expiresIn: '24h', algorithm: 'HS256' }
  );

  // SECURITY: Headers de seguridad
  res.setHeader('X-Content-Type-Options', 'nosniff');
  res.setHeader('X-Frame-Options', 'DENY');

  return res.json({
    token,
    user: {
      id: user.id,
      email: user.email,
      name: user.name,
    },
  });
}

// ============================================================
// DESPUÉS: Refactorizado manteniendo seguridad
// ============================================================

import bcrypt from 'bcrypt';
import jwt from 'jsonwebtoken';
import rateLimit from 'express-rate-limit';

// SECURITY: Rate limiting - NO simplificar configuración
const loginRateLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,
  message: 'Too many login attempts',
  standardHeaders: true,
  legacyHeaders: false,
});

// SECURITY: Constantes de seguridad extraídas pero preservadas
const LOCKOUT_DURATION_MS = 15 * 60 * 1000;
const MAX_FAILED_ATTEMPTS = 5;
const JWT_EXPIRES_IN = '24h';

// Extraído: Validación de input
function validateLoginInput(body: unknown): { email: string; password: string } {
  if (!body || typeof body !== 'object') {
    throw new ValidationError('Invalid request body');
  }

  const { email, password } = body as Record<string, unknown>;

  if (!email || !password) {
    throw new ValidationError('Email and password required');
  }

  if (typeof email !== 'string' || typeof password !== 'string') {
    throw new ValidationError('Invalid credential types');
  }

  // SECURITY: Validación de formato de email
  if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
    throw new ValidationError('Invalid email format');
  }

  // SECURITY: Validación de longitud de contraseña
  if (password.length < 8 || password.length > 128) {
    throw new ValidationError('Invalid password length');
  }

  return { email, password };
}

// Extraído: Verificación de cuenta bloqueada
function checkAccountLockout(user: User): void {
  if (user.lockedUntil && user.lockedUntil > new Date()) {
    const retryAfter = Math.ceil((user.lockedUntil.getTime() - Date.now()) / 1000);
    throw new AccountLockedError(retryAfter);
  }
}

// Extraído: Manejo de login fallido
async function handleFailedLogin(user: User): Promise<void> {
  user.failedLoginAttempts = (user.failedLoginAttempts || 0) + 1;

  if (user.failedLoginAttempts >= MAX_FAILED_ATTEMPTS) {
    user.lockedUntil = new Date(Date.now() + LOCKOUT_DURATION_MS);
  }

  await user.save();
}

// Extraído: Manejo de login exitoso
async function handleSuccessfulLogin(user: User): Promise<string> {
  user.failedLoginAttempts = 0;
  user.lockedUntil = null;
  user.lastLoginAt = new Date();
  await user.save();

  // SECURITY: JWT con expiración y algoritmo explícitos
  return jwt.sign(
    { userId: user.id, email: user.email },
    process.env.JWT_SECRET!,
    { expiresIn: JWT_EXPIRES_IN, algorithm: 'HS256' }
  );
}

// SECURITY: Función principal - se permite >20 líneas
async function handleLogin(req: Request, res: Response) {
  try {
    // Validar input
    const { email, password } = validateLoginInput(req.body);

    // Buscar usuario
    const user = await db.users.findOne({ where: { email } });

    if (!user) {
      // SECURITY: Timing-safe dummy comparison
      await bcrypt.compare('dummy', '$2b$10$dummyhashfordummyuser');
      throw new AuthenticationError('Invalid credentials');
    }

    // Verificar bloqueo
    checkAccountLockout(user);

    // Verificar contraseña
    const isValid = await bcrypt.compare(password, user.passwordHash);

    if (!isValid) {
      await handleFailedLogin(user);
      throw new AuthenticationError('Invalid credentials');
    }

    // Login exitoso
    const token = await handleSuccessfulLogin(user);

    // SECURITY: Headers de seguridad - preservados
    res.setHeader('X-Content-Type-Options', 'nosniff');
    res.setHeader('X-Frame-Options', 'DENY');

    return res.json({
      token,
      user: {
        id: user.id,
        email: user.email,
        name: user.name,
      },
    });

  } catch (error) {
    if (error instanceof ValidationError) {
      return res.status(400).json({ error: error.message });
    }
    if (error instanceof AccountLockedError) {
      return res.status(423).json({
        error: 'Account locked',
        retryAfter: error.retryAfter,
      });
    }
    if (error instanceof AuthenticationError) {
      return res.status(401).json({ error: error.message });
    }
    throw error;
  }
}

// ============================================================
// REPORTE DE CAMBIOS
// ============================================================

// [KISS] Dividida función God de 45 líneas en 5 funciones
// [KISS] Aplicadas guard clauses en validación
// [KISS] Extraídas constantes de configuración
// [DRY] Centralizada lógica de manejo de login fallido/exitoso
// [SECURITY] Preservado timing-safe comparison para usuarios no existentes
// [SECURITY] Preservada validación de formato de email
// [SECURITY] Preservada validación de longitud de contraseña
// [SECURITY] Preservados headers de seguridad
// [SECURITY] Preservada configuración de rate limiting
// [SECURITY] Preservado manejo de cuenta bloqueada
// [SECURITY] Preservada generación de JWT con parámetros explícitos
```

---

## Resumen de Protecciones

| Mecanismo | Protege Contra | Cómo Funciona |
|-----------|----------------|---------------|
| **Detector de Validaciones** | Eliminación accidental de validaciones | Patrones heurísticos en nombres, imports, regex |
| **Reglas Diferenciadas** | Simplificación excesiva | Límites más altos para código de seguridad |
| **Lista Blanca** | Eliminación de controles críticos | Patrones que nunca se tocan (hashing, CSRF, etc.) |
| **Marcadores** | Pérdida de intención de seguridad | Comentarios y convenciones de nombres |
| **Verificación Post** | Regresiones de seguridad | Checks automáticos después de cambios |
| **Análisis de Flujo** | Exposición de datos sensibles | Rastreo de datos sensibles |
| **Contexto por Dominio** | Reglas inapropiadas | Reglas específicas por tipo de archivo |

---

## Principio Fundamental

> **"La simplicidad nunca debe sacrificar la seguridad. Un código simple pero inseguro es peor que código complejo pero seguro."**

Estos mecanismos aseguran que la skill KISS-DRY-YAGNI sea:
- **Afilada como navaja** para complejidad innecesaria
- **Cirujana precisa** con código de seguridad
