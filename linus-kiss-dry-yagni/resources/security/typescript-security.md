# Seguridad Específica - TypeScript/JavaScript

> Guía de protecciones de seguridad para código TypeScript/JavaScript (Node.js/Browser). NUNCA eliminar estos patrones al aplicar KISS-DRY-YAGNI.

---

## 1. Eval y Ejecución Dinámica

### El Problema
```typescript
// ❌ PROHIBIDO - NUNCA con input del usuario
const userInput = req.body.code;
eval(userInput); // RCE total

// ❌ PROHIBIDO - Function constructor
const fn = new Function('return ' + userInput)();

// ❌ PROHIBIDO - setTimeout/setInterval con strings
setTimeout(userInput, 1000);
```

### La Solución
```typescript
// ✅ CORRECTO - JSON.parse para datos estructurados
const data = JSON.parse(userInput);

// ✅ CORRECTO - jsep o similar para expresiones matemáticas simples
import * as jsep from 'jsep';

function evaluateMath(expression: string): number {
    const allowed = /^[0-9+\-*/.()\s]+$/;
    if (!allowed.test(expression)) {
        throw new Error('Invalid characters');
    }

    const ast = jsep(expression);
    // Evaluar AST de forma controlada
    return evaluateNode(ast);
}

// ✅ CORRECTO - VM2 o isolated-vm para sandboxing (si es inevitable)
import { VM } from 'vm2';

const vm = new VM({
    timeout: 1000,
    sandbox: { /* only expose safe objects */ }
});

const result = vm.run('1 + 1'); // Más seguro que eval
```

---

## 2. Prototype Pollution

```typescript
// ❌ PROHIBIDO - Merge recursivo sin validación
function merge(target: any, source: any) {
    for (const key in source) {
        if (typeof source[key] === 'object') {
            merge(target[key], source[key]);
        } else {
            target[key] = source[key];
        }
    }
}

// Ataque: merge({}, JSON.parse('{"__proto__": {"isAdmin": true}}'))

// ✅ CORRECTO - Usar Object.assign o structuredClone
const merged = Object.assign({}, target, source);

// ✅ CORRECTO - Validar keys prohibidas
const FORBIDDEN_KEYS = ['__proto__', 'constructor', 'prototype'];

function safeMerge(target: any, source: any): any {
    for (const key in source) {
        if (FORBIDDEN_KEYS.includes(key)) {
            throw new Error(`Forbidden key: ${key}`);
        }

        if (typeof source[key] === 'object' && source[key] !== null) {
            target[key] = safeMerge(target[key] || {}, source[key]);
        } else {
            target[key] = source[key];
        }
    }
    return target;
}

// ✅ CORRECTO - Usar librerías seguras como lodash con fp
import { merge } from 'lodash/fp';
const merged = merge(target, source); // Safe
```

---

## 3. SQL Injection

```typescript
// ❌ PROHIBIDO - Template literals con variables
const query = `SELECT * FROM users WHERE id = ${userId}`;

// ❌ PROHIBIDO - Concatenación de strings
const query = "SELECT * FROM users WHERE id = '" + userId + "'";

// ✅ CORRECTO - mysql2 con placeholders
import mysql from 'mysql2/promise';
const [rows] = await connection.execute(
    'SELECT * FROM users WHERE id = ? AND active = ?',
    [userId, true]
);

// ✅ CORRECTO - pg (PostgreSQL) con parametrización
import { Pool } from 'pg';
const result = await pool.query(
    'SELECT * FROM users WHERE id = $1',
    [userId]
);

// ✅ CORRECTO - Prisma ORM
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

const user = await prisma.user.findUnique({
    where: { id: userId }
});

// ✅ CORRECTO - TypeORM
import { Repository } from 'typeorm';
const userRepo: Repository<User> = dataSource.getRepository(User);
const user = await userRepo.findOneBy({ id: userId });

// ✅ CORRECTO - Knex.js query builder
import knex from 'knex';
const users = await knex('users')
    .where({ id: userId, active: true })
    .select('*');
```

---

## 4. Path Traversal

```typescript
// ❌ PROHIBIDO - Path sin sanitización
const filePath = path.join('./uploads', req.query.filename);
fs.readFileSync(filePath); // Ataque: ../../../etc/passwd

// ✅ CORRECTO - Validar path resuelto
import path from 'path';
import { promises as fs } from 'fs';

const UPLOAD_DIR = path.resolve('./uploads');

async function readUserFile(filename: string): Promise<Buffer> {
    // Normalizar y resolver path
    const filePath = path.resolve(UPLOAD_DIR, filename);

    // Verificar que está dentro del directorio permitido
    if (!filePath.startsWith(UPLOAD_DIR)) {
        throw new Error('Path traversal detected');
    }

    // Verificar que no contiene caracteres especiales
    const basename = path.basename(filename);
    if (!/^[a-zA-Z0-9._-]+$/.test(basename)) {
        throw new Error('Invalid filename');
    }

    return fs.readFile(filePath);
}

// ✅ CORRECTO - UUID como nombre de archivo
import { v4 as uuidv4 } from 'uuid';

async function saveUpload(data: Buffer, originalName: string): Promise<string> {
    const ext = path.extname(originalName).toLowerCase();
    const allowedExts = ['.jpg', '.png', '.pdf'];

    if (!allowedExts.includes(ext)) {
        throw new Error('Invalid extension');
    }

    const filename = `${uuidv4()}${ext}`;
    const filePath = path.join(UPLOAD_DIR, filename);

    await fs.writeFile(filePath, data);
    return filename;
}
```

---

## 5. Deserialización Segura

```typescript
// ❌ PROHIBIDO - eval para "JSON" (¡sí, existe!)
const data = eval('(' + userInput + ')');

// ✅ CORRECTO - JSON.parse
const data = JSON.parse(userInput);

// ✅ CORRECTO - Validación con Zod
import { z } from 'zod';

const UserSchema = z.object({
    id: z.number().int().positive(),
    name: z.string().min(1).max(100),
    email: z.string().email(),
    age: z.number().int().min(0).max(150).optional(),
});

type User = z.infer<typeof UserSchema>;

function parseUser(data: unknown): User {
    return UserSchema.parse(data);
}

// ✅ CORRECTO - class-validator (TypeORM/NestJS)
import { IsEmail, IsInt, IsString, Length, Min, Max } from 'class-validator';

class CreateUserDto {
    @IsString()
    @Length(3, 50)
    name: string;

    @IsEmail()
    email: string;

    @IsInt()
    @Min(0)
    @Max(150)
    age: number;
}

// ✅ CORRECTO - joi
import Joi from 'joi';

const schema = Joi.object({
    username: Joi.string().alphanum().min(3).max(30).required(),
    password: Joi.string().pattern(new RegExp('^[a-zA-Z0-9]{3,30}$')),
    email: Joi.string().email({ minDomainSegments: 2 })
});
```

---

## 6. Secrets Management

```typescript
// ❌ PROHIBIDO - Hardcodear secrets
const API_KEY = 'sk_live_1234567890abcdef';

// ✅ CORRECTO - Variables de entorno
const API_KEY = process.env.API_KEY;
if (!API_KEY) {
    throw new Error('API_KEY environment variable is required');
}

// ✅ CORRECTO - dotenv para desarrollo
import dotenv from 'dotenv';
dotenv.config();

const config = {
    dbUrl: process.env.DATABASE_URL!,
    jwtSecret: process.env.JWT_SECRET!,
    apiKey: process.env.API_KEY!,
};

// Validar en startup
function validateConfig() {
    const required = ['DATABASE_URL', 'JWT_SECRET', 'API_KEY'];
    for (const key of required) {
        if (!process.env[key]) {
            throw new Error(`Missing required environment variable: ${key}`);
        }
    }
}

// ✅ CORRECTO - AWS Secrets Manager / Azure Key Vault
import { SecretsManagerClient, GetSecretValueCommand } from '@aws-sdk/client-secrets-manager';

async function getSecret(secretName: string): Promise<string> {
    const client = new SecretsManagerClient({ region: 'us-east-1' });
    const command = new GetSecretValueCommand({ SecretId: secretName });
    const response = await client.send(command);
    return response.SecretString!;
}
```

---

## 7. Logging Seguro

```typescript
// ❌ PROHIBIDO - Loggear datos sensibles
logger.info('User login', { email, password, creditCard });

// ✅ CORRECTO - Sanitizar antes de loggear
import winston from 'winston';

const SENSITIVE_FIELDS = [
    'password', 'token', 'secret', 'apiKey', 'creditCard',
    'ssn', 'cvv', 'authorization'
];

function sanitizeForLogging(obj: any): any {
    if (typeof obj !== 'object' || obj === null) {
        return obj;
    }

    const sanitized = { ...obj };
    for (const key of Object.keys(sanitized)) {
        const lowerKey = key.toLowerCase();
        if (SENSITIVE_FIELDS.some(f => lowerKey.includes(f))) {
            sanitized[key] = '***REDACTED***';
        } else if (typeof sanitized[key] === 'object') {
            sanitized[key] = sanitizeForLogging(sanitized[key]);
        }
    }
    return sanitized;
}

logger.info('User login', sanitizeForLogging({ email, password }));

// ✅ CORRECTO - Winston con redaction
const logger = winston.createLogger({
    format: winston.format.combine(
        winston.format((info) => {
            return sanitizeForLogging(info);
        })(),
        winston.format.json()
    ),
    transports: [new winston.transports.Console()]
});
```

---

## 8. Express.js Seguro

```typescript
import express from 'express';
import helmet from 'helmet';
import rateLimit from 'express-rate-limit';
import cors from 'cors';
import mongoSanitize from 'express-mongo-sanitize';
import hpp from 'hpp';

const app = express();

// ✅ CORRECTO - Helmet para headers de seguridad
app.use(helmet({
    contentSecurityPolicy: {
        directives: {
            defaultSrc: ["'self'"],
            styleSrc: ["'self'", "'unsafe-inline'"],
            scriptSrc: ["'self'"],
            imgSrc: ["'self'", "data:", "https:"],
        },
    },
    hsts: {
        maxAge: 31536000,
        includeSubDomains: true,
        preload: true
    }
}));

// ✅ CORRECTO - CORS restrictivo (NO '*')
app.use(cors({
    origin: ['https://app.example.com', 'https://admin.example.com'],
    methods: ['GET', 'POST', 'PUT', 'DELETE'],
    allowedHeaders: ['Content-Type', 'Authorization'],
    credentials: true,
    maxAge: 86400
}));

// ✅ CORRECTO - Rate limiting
const limiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutos
    max: 100, // limitar cada IP a 100 requests por ventana
    message: 'Too many requests from this IP',
    standardHeaders: true,
    legacyHeaders: false,
});
app.use('/api/', limiter);

// Rate limiting más estricto para auth
const authLimiter = rateLimit({
    windowMs: 15 * 60 * 1000,
    max: 5,
    skipSuccessfulRequests: true, // No contar logins exitosos
});
app.use('/api/auth/login', authLimiter);

// ✅ CORRECTO - Body parsing con límites
app.use(express.json({ limit: '10kb' }));
app.use(express.urlencoded({ extended: true, limit: '10kb' }));

// ✅ CORRECTO - Sanitización NoSQL injection
app.use(mongoSanitize());

// ✅ CORRECTO - Prevención Parameter Pollution
app.use(hpp());
```

---

## 9. Password Hashing

```typescript
// ✅ CORRECTO - bcrypt
import bcrypt from 'bcrypt';

const SALT_ROUNDS = 12;

async function hashPassword(password: string): Promise<string> {
    return bcrypt.hash(password, SALT_ROUNDS);
}

async function verifyPassword(password: string, hash: string): Promise<boolean> {
    return bcrypt.compare(password, hash);
}

// ✅ CORRECTO - argon2 (más moderno)
import argon2 from 'argon2';

async function hashPasswordArgon(password: string): Promise<string> {
    return argon2.hash(password, {
        type: argon2.argon2id,
        memoryCost: 65536, // 64MB
        timeCost: 3,
        parallelism: 4
    });
}

async function verifyPasswordArgon(password: string, hash: string): Promise<boolean> {
    return argon2.verify(hash, password);
}

// ✅ CORRECTO - PBKDF2 para derivación de claves
import crypto from 'crypto';

function deriveKey(password: string, salt: Buffer): Buffer {
    return crypto.pbkdf2Sync(password, salt, 100000, 64, 'sha512');
}
```

---

## 10. JWT Seguro

```typescript
import jwt from 'jsonwebtoken';
import { Request, Response, NextFunction } from 'express';

const JWT_SECRET = process.env.JWT_SECRET!;
const JWT_EXPIRES_IN = '24h';

interface TokenPayload {
    userId: string;
    email: string;
    type: 'access' | 'refresh';
}

// ✅ CORRECTO - Crear token con claims explícitas
export function createToken(payload: Omit<TokenPayload, 'type'>): string {
    return jwt.sign(
        {
            userId: payload.userId,
            email: payload.email,
            type: 'access',
            iat: Math.floor(Date.now() / 1000),
        },
        JWT_SECRET,
        {
            expiresIn: JWT_EXPIRES_IN,
            algorithm: 'HS256', // Especificar algoritmo
            issuer: 'myapp',
            audience: 'myapp-client'
        }
    );
}

// ✅ CORRECTO - Verificar token estrictamente
export function verifyToken(token: string): TokenPayload {
    try {
        const decoded = jwt.verify(token, JWT_SECRET, {
            algorithms: ['HS256'], // Solo aceptar HS256
            issuer: 'myapp',
            audience: 'myapp-client',
            complete: false
        }) as TokenPayload;

        if (decoded.type !== 'access') {
            throw new Error('Invalid token type');
        }

        return decoded;
    } catch (error) {
        if (error instanceof jwt.TokenExpiredError) {
            throw new Error('Token expired');
        }
        if (error instanceof jwt.JsonWebTokenError) {
            throw new Error('Invalid token');
        }
        throw error;
    }
}

// ✅ CORRECTO - Middleware de autenticación
export function authenticateToken(req: Request, res: Response, next: NextFunction) {
    const authHeader = req.headers['authorization'];
    const token = authHeader && authHeader.split(' ')[1]; // Bearer TOKEN

    if (!token) {
        return res.status(401).json({ error: 'Access token required' });
    }

    try {
        const payload = verifyToken(token);
        req.user = payload;
        next();
    } catch (error) {
        return res.status(403).json({ error: 'Invalid or expired token' });
    }
}
```

---

## 11. Validación de Uploads

```typescript
import multer from 'multer';
import path from 'path';
import { v4 as uuidv4 } from 'uuid';
import fileType from 'file-type';
import fs from 'fs/promises';

// ✅ CORRECTO - Configuración segura de multer
const storage = multer.memoryStorage(); // No guardar en disco inmediatamente

const fileFilter = (req: any, file: Express.Multer.File, cb: multer.FileFilterCallback) => {
    const allowedMimes = ['image/jpeg', 'image/png', 'application/pdf'];
    if (allowedMimes.includes(file.mimetype)) {
        cb(null, true);
    } else {
        cb(new Error('Invalid file type'));
    }
};

const upload = multer({
    storage,
    limits: {
        fileSize: 10 * 1024 * 1024, // 10MB
        files: 1
    },
    fileFilter
});

// ✅ CORRECTO - Validación de magic bytes
async function validateFile(file: Express.Multer.File): Promise<void> {
    // Validar tamaño
    if (file.size > 10 * 1024 * 1024) {
        throw new Error('File too large');
    }

    // Detectar tipo real por magic bytes
    const type = await fileType.fromBuffer(file.buffer);
    if (!type) {
        throw new Error('Could not determine file type');
    }

    const allowedTypes = ['image/jpeg', 'image/png', 'application/pdf'];
    if (!allowedTypes.includes(type.mime)) {
        throw new Error('File type not allowed');
    }

    // Verificar que MIME declarado coincide con real
    if (file.mimetype !== type.mime) {
        throw new Error('MIME type mismatch');
    }
}

// ✅ CORRECTO - Guardar con nombre aleatorio
async function saveFile(file: Express.Multer.File): Promise<string> {
    const ext = path.extname(file.originalname).toLowerCase();
    const filename = `${uuidv4()}${ext}`;
    const filepath = path.join('./uploads', filename);

    await fs.writeFile(filepath, file.buffer);
    return filename;
}
```

---

## 12. XSS Prevention

```typescript
// ❌ PROHIBIDO - InnerHTML con input del usuario
element.innerHTML = userInput;

// ✅ CORRECTO - textContent
const element = document.createElement('div');
element.textContent = userInput;

// ✅ CORRECTO - DOMPurify para HTML
import DOMPurify from 'isomorphic-dompurify';

const clean = DOMPurify.sanitize(dirtyHtml, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a'],
    ALLOWED_ATTR: ['href']
});

// ✅ CORRECTO - Escape en templates
import escapeHtml from 'escape-html';

const html = `<div>${escapeHtml(userInput)}</div>`;

// ✅ CORRECTO - React (auto-escape por defecto)
function SafeComponent({ userInput }: { userInput: string }) {
    return <div>{userInput}</div>; // Escapado automático
}

// ❌ En React: dangerouslySetInnerHTML
function UnsafeComponent({ html }: { html: string }) {
    return <div dangerouslySetInnerHTML={{ __html: html }} />; // ⚠️ Peligroso
}

// ✅ Seguro en React:
import DOMPurify from 'dompurify';

function SafeHtmlComponent({ html }: { html: string }) {
    const clean = DOMPurify.sanitize(html);
    return <div dangerouslySetInnerHTML={{ __html: clean }} />;
}
```

---

## Checklist TypeScript Específico

### Seguridad General
- [ ] NUNCA usar eval() o new Function() con input externo
- [ ] NUNCA usar innerHTML con input no sanitizado
- [ ] Proteger contra prototype pollution (validar keys)
- [ ] SQL parametrizado (?, $1, no concatenación)
- [ ] Validar paths de archivos (evitar ../)
- [ ] Validar deserialización con Zod/Joi/class-validator
- [ ] Sanitizar logs (no passwords/tokens)
- [ ] Secrets en variables de entorno
- [ ] Helmet para headers de seguridad
- [ ] Rate limiting en endpoints sensibles
- [ ] CORS restrictivo (no '*')
- [ ] bcrypt/argon2 para passwords
- [ ] JWT con verificación estricta de algoritmo
- [ ] DOMPurify para HTML dinámico

### Express/Node.js
- [ ] Body parser con límites de tamaño
- [ ] mongoSanitize para NoSQL injection
- [ ] hpp para parameter pollution
- [ ] Graceful shutdown con SIGTERM/SIGINT handlers
- [ ] Timeouts en conexiones DB
- [ ] Validación de uploads (tamaño, tipo, magic bytes)

### Frontend
- [ ] Content Security Policy
- [ ] Subresource Integrity (SRI)
- [ ] HttpOnly cookies para tokens
- [ ] Sanitización de URLs (javascript: protocol)

---

## Referencias

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Node.js Security Checklist](https://blog.risingstack.com/node-js-security-checklist/)
- [Express.js Security Best Practices](https://expressjs.com/en/advanced/best-practice-security.html)
- [Snyk JavaScript Security](https://snyk.io/learn/javascript-security/)
