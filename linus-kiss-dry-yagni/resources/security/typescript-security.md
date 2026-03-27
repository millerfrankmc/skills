# Language-Specific Security - TypeScript/JavaScript

> Security protection guide for TypeScript/JavaScript code (Node.js/Browser). NEVER remove these patterns when applying KISS-DRY-YAGNI.

---

## 1. Eval and Dynamic Execution

### The Problem
```typescript
// ❌ FORBIDDEN - NEVER with user input
const userInput = req.body.code;
eval(userInput); // Total RCE

// ❌ FORBIDDEN - Function constructor
const fn = new Function('return ' + userInput)();

// ❌ FORBIDDEN - setTimeout/setInterval with strings
setTimeout(userInput, 1000);
```

### The Solution
```typescript
// ✅ CORRECT - JSON.parse for structured data
const data = JSON.parse(userInput);

// ✅ CORRECT - jsep or similar for simple math expressions
import * as jsep from 'jsep';

function evaluateMath(expression: string): number {
    const allowed = /^[0-9+\-*/.()\s]+$/;
    if (!allowed.test(expression)) {
        throw new Error('Invalid characters');
    }

    const ast = jsep(expression);
    // Evaluate AST in controlled way
    return evaluateNode(ast);
}

// ✅ CORRECT - VM2 or isolated-vm for sandboxing (if inevitable)
import { VM } from 'vm2';

const vm = new VM({
    timeout: 1000,
    sandbox: { /* only expose safe objects */ }
});

const result = vm.run('1 + 1'); // Safer than eval
```

---

## 2. Prototype Pollution

```typescript
// ❌ FORBIDDEN - Recursive merge without validation
function merge(target: any, source: any) {
    for (const key in source) {
        if (typeof source[key] === 'object') {
            merge(target[key], source[key]);
        } else {
            target[key] = source[key];
        }
    }
}

// Attack: merge({}, JSON.parse('{"__proto__": {"isAdmin": true}}'))

// ✅ CORRECT - Use Object.assign or structuredClone
const merged = Object.assign({}, target, source);

// ✅ CORRECT - Validate forbidden keys
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

// ✅ CORRECT - Use safe libraries like lodash with fp
import { merge } from 'lodash/fp';
const merged = merge(target, source); // Safe
```

---

## 3. SQL Injection

```typescript
// ❌ FORBIDDEN - Template literals with variables
const query = `SELECT * FROM users WHERE id = ${userId}`;

// ❌ FORBIDDEN - String concatenation
const query = "SELECT * FROM users WHERE id = '" + userId + "'";

// ✅ CORRECT - mysql2 with placeholders
import mysql from 'mysql2/promise';
const [rows] = await connection.execute(
    'SELECT * FROM users WHERE id = ? AND active = ?',
    [userId, true]
);

// ✅ CORRECT - pg (PostgreSQL) with parameterization
import { Pool } from 'pg';
const result = await pool.query(
    'SELECT * FROM users WHERE id = $1',
    [userId]
);

// ✅ CORRECT - Prisma ORM
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

const user = await prisma.user.findUnique({
    where: { id: userId }
});

// ✅ CORRECT - TypeORM
import { Repository } from 'typeorm';
const userRepo: Repository<User> = dataSource.getRepository(User);
const user = await userRepo.findOneBy({ id: userId });

// ✅ CORRECT - Knex.js query builder
import knex from 'knex';
const users = await knex('users')
    .where({ id: userId, active: true })
    .select('*');
```

---

## 4. Path Traversal

```typescript
// ❌ FORBIDDEN - Path without sanitization
const filePath = path.join('./uploads', req.query.filename);
fs.readFileSync(filePath); // Attack: ../../../etc/passwd

// ✅ CORRECT - Validate resolved path
import path from 'path';
import { promises as fs } from 'fs';

const UPLOAD_DIR = path.resolve('./uploads');

async function readUserFile(filename: string): Promise<Buffer> {
    // Normalize and resolve path
    const filePath = path.resolve(UPLOAD_DIR, filename);

    // Verify it's within the allowed directory
    if (!filePath.startsWith(UPLOAD_DIR)) {
        throw new Error('Path traversal detected');
    }

    // Verify it doesn't contain special characters
    const basename = path.basename(filename);
    if (!/^[a-zA-Z0-9._-]+$/.test(basename)) {
        throw new Error('Invalid filename');
    }

    return fs.readFile(filePath);
}

// ✅ CORRECT - UUID as filename
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

## 5. Safe Deserialization

```typescript
// ❌ FORBIDDEN - eval for "JSON" (yes, it exists!)
const data = eval('(' + userInput + ')');

// ✅ CORRECT - JSON.parse
const data = JSON.parse(userInput);

// ✅ CORRECT - Validation with Zod
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

// ✅ CORRECT - class-validator (TypeORM/NestJS)
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

// ✅ CORRECT - joi
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
// ❌ FORBIDDEN - Hardcoding secrets
const API_KEY = 'sk_live_1234567890abcdef';

// ✅ CORRECT - Environment variables
const API_KEY = process.env.API_KEY;
if (!API_KEY) {
    throw new Error('API_KEY environment variable is required');
}

// ✅ CORRECT - dotenv for development
import dotenv from 'dotenv';
dotenv.config();

const config = {
    dbUrl: process.env.DATABASE_URL!,
    jwtSecret: process.env.JWT_SECRET!,
    apiKey: process.env.API_KEY!,
};

// Validate at startup
function validateConfig() {
    const required = ['DATABASE_URL', 'JWT_SECRET', 'API_KEY'];
    for (const key of required) {
        if (!process.env[key]) {
            throw new Error(`Missing required environment variable: ${key}`);
        }
    }
}

// ✅ CORRECT - AWS Secrets Manager / Azure Key Vault
import { SecretsManagerClient, GetSecretValueCommand } from '@aws-sdk/client-secrets-manager';

async function getSecret(secretName: string): Promise<string> {
    const client = new SecretsManagerClient({ region: 'us-east-1' });
    const command = new GetSecretValueCommand({ SecretId: secretName });
    const response = await client.send(command);
    return response.SecretString!;
}
```

---

## 7. Secure Logging

```typescript
// ❌ FORBIDDEN - Logging sensitive data
logger.info('User login', { email, password, creditCard });

// ✅ CORRECT - Sanitize before logging
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

// ✅ CORRECT - Winston with redaction
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

## 8. Secure Express.js

```typescript
import express from 'express';
import helmet from 'helmet';
import rateLimit from 'express-rate-limit';
import cors from 'cors';
import mongoSanitize from 'express-mongo-sanitize';
import hpp from 'hpp';

const app = express();

// ✅ CORRECT - Helmet for security headers
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

// ✅ CORRECT - Restrictive CORS (NOT '*')
app.use(cors({
    origin: ['https://app.example.com', 'https://admin.example.com'],
    methods: ['GET', 'POST', 'PUT', 'DELETE'],
    allowedHeaders: ['Content-Type', 'Authorization'],
    credentials: true,
    maxAge: 86400
}));

// ✅ CORRECT - Rate limiting
const limiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 100, // limit each IP to 100 requests per window
    message: 'Too many requests from this IP',
    standardHeaders: true,
    legacyHeaders: false,
});
app.use('/api/', limiter);

// Stricter rate limiting for auth
const authLimiter = rateLimit({
    windowMs: 15 * 60 * 1000,
    max: 5,
    skipSuccessfulRequests: true, // Don't count successful logins
});
app.use('/api/auth/login', authLimiter);

// ✅ CORRECT - Body parsing with limits
app.use(express.json({ limit: '10kb' }));
app.use(express.urlencoded({ extended: true, limit: '10kb' }));

// ✅ CORRECT - NoSQL injection sanitization
app.use(mongoSanitize());

// ✅ CORRECT - Parameter Pollution Prevention
app.use(hpp());
```

---

## 9. Password Hashing

```typescript
// ✅ CORRECT - bcrypt
import bcrypt from 'bcrypt';

const SALT_ROUNDS = 12;

async function hashPassword(password: string): Promise<string> {
    return bcrypt.hash(password, SALT_ROUNDS);
}

async function verifyPassword(password: string, hash: string): Promise<boolean> {
    return bcrypt.compare(password, hash);
}

// ✅ CORRECT - argon2 (more modern)
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

// ✅ CORRECT - PBKDF2 for key derivation
import crypto from 'crypto';

function deriveKey(password: string, salt: Buffer): Buffer {
    return crypto.pbkdf2Sync(password, salt, 100000, 64, 'sha512');
}
```

---

## 10. Secure JWT

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

// ✅ CORRECT - Create token with explicit claims
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
            algorithm: 'HS256', // Specify algorithm
            issuer: 'myapp',
            audience: 'myapp-client'
        }
    );
}

// ✅ CORRECT - Verify token strictly
export function verifyToken(token: string): TokenPayload {
    try {
        const decoded = jwt.verify(token, JWT_SECRET, {
            algorithms: ['HS256'], // Only accept HS256
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

// ✅ CORRECT - Authentication middleware
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

## 11. Upload Validation

```typescript
import multer from 'multer';
import path from 'path';
import { v4 as uuidv4 } from 'uuid';
import fileType from 'file-type';
import fs from 'fs/promises';

// ✅ CORRECT - Secure multer configuration
const storage = multer.memoryStorage(); // Don't save to disk immediately

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

// ✅ CORRECT - Magic bytes validation
async function validateFile(file: Express.Multer.File): Promise<void> {
    // Validate size
    if (file.size > 10 * 1024 * 1024) {
        throw new Error('File too large');
    }

    // Detect real type by magic bytes
    const type = await fileType.fromBuffer(file.buffer);
    if (!type) {
        throw new Error('Could not determine file type');
    }

    const allowedTypes = ['image/jpeg', 'image/png', 'application/pdf'];
    if (!allowedTypes.includes(type.mime)) {
        throw new Error('File type not allowed');
    }

    // Verify declared MIME matches real
    if (file.mimetype !== type.mime) {
        throw new Error('MIME type mismatch');
    }
}

// ✅ CORRECT - Save with random name
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
// ❌ FORBIDDEN - InnerHTML with user input
element.innerHTML = userInput;

// ✅ CORRECT - textContent
const element = document.createElement('div');
element.textContent = userInput;

// ✅ CORRECT - DOMPurify for HTML
import DOMPurify from 'isomorphic-dompurify';

const clean = DOMPurify.sanitize(dirtyHtml, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a'],
    ALLOWED_ATTR: ['href']
});

// ✅ CORRECT - Escape in templates
import escapeHtml from 'escape-html';

const html = `<div>${escapeHtml(userInput)}</div>`;

// ✅ CORRECT - React (auto-escape by default)
function SafeComponent({ userInput }: { userInput: string }) {
    return <div>{userInput}</div>; // Automatic escaping
}

// ❌ In React: dangerouslySetInnerHTML
function UnsafeComponent({ html }: { html: string }) {
    return <div dangerouslySetInnerHTML={{ __html: html }} />; // ⚠️ Dangerous
}

// ✅ Safe in React:
import DOMPurify from 'dompurify';

function SafeHtmlComponent({ html }: { html: string }) {
    const clean = DOMPurify.sanitize(html);
    return <div dangerouslySetInnerHTML={{ __html: clean }} />;
}
```

---

## TypeScript-Specific Checklist

### General Security
- [ ] NEVER use eval() or new Function() with external input
- [ ] NEVER use innerHTML with unsanitized input
- [ ] Protect against prototype pollution (validate keys)
- [ ] Parameterized SQL (?, $1, no concatenation)
- [ ] Validate file paths (prevent ../)
- [ ] Validate deserialization with Zod/Joi/class-validator
- [ ] Sanitize logs (no passwords/tokens)
- [ ] Secrets in environment variables
- [ ] Helmet for security headers
- [ ] Rate limiting on sensitive endpoints
- [ ] Restrictive CORS (no '*')
- [ ] bcrypt/argon2 for passwords
- [ ] JWT with strict algorithm verification
- [ ] DOMPurify for dynamic HTML

### Express/Node.js
- [ ] Body parser with size limits
- [ ] mongoSanitize for NoSQL injection
- [ ] hpp for parameter pollution
- [ ] Graceful shutdown with SIGTERM/SIGINT handlers
- [ ] Timeouts on DB connections
- [ ] Upload validation (size, type, magic bytes)

### Frontend
- [ ] Content Security Policy
- [ ] Subresource Integrity (SRI)
- [ ] HttpOnly cookies for tokens
- [ ] URL sanitization (javascript: protocol)

---

## References

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Node.js Security Checklist](https://blog.risingstack.com/node-js-security-checklist/)
- [Express.js Security Best Practices](https://expressjs.com/en/advanced/best-practice-security.html)
- [Snyk JavaScript Security](https://snyk.io/learn/javascript-security/)
