# Language-Specific Security - Python

> Guide for security protections in Python code. NEVER remove these patterns when applying KISS-DRY-YAGNI.

---

## 1. Safe Deserialization

### The Problem: pickle is dangerous
```python
# ❌ FORBIDDEN - NEVER use pickle with untrusted data
import pickle

def process_request(data):
    obj = pickle.loads(data)  # RCE if data is malicious
    return obj
```

### The Solution
```python
# ✅ CORRECT - Use JSON for untrusted data
import json

def process_request(data: bytes) -> dict:
    return json.loads(data)  # Safe, does not execute code

# ✅ If you need pickle (internal data), validate source
def process_internal_data(data: bytes, source: str) -> Any:
    if source not in TRUSTED_INTERNAL_SERVICES:
        raise SecurityError("Untrusted source")
    return pickle.loads(data)
```

### Safe Alternatives
```python
# JSON - For client/API data
import json

# MessagePack - Binary but safe
import msgpack

# Protobuf - Strict and efficient
from google.protobuf import message

# Safe YAML - NEVER use load(), only safe_load()
import yaml
data = yaml.safe_load(stream)  # ✅
data = yaml.load(stream)       # ❌ Unsafe by default
```

---

## 2. Code Injection

### eval() and exec()
```python
# ❌ FORBIDDEN - NEVER use with user input
def calculate(expression: str) -> Any:
    return eval(expression)  # Full RCE

# ✅ CORRECT - Use safe_eval or specialized libraries
import ast
import operator

ALLOWED_OPERATORS = {
    ast.Add: operator.add,
    ast.Sub: operator.sub,
    ast.Mult: operator.mul,
    ast.Div: operator.truediv,
}

def safe_eval(expression: str) -> float:
    tree = ast.parse(expression, mode='eval')
    if not all(isinstance(node, (ast.Expression, ast.BinOp, ast.Num))
               for node in ast.walk(tree)):
        raise ValueError("Unsafe expression")
    return eval(compile(tree, '', 'eval'), {"__builtins__": {}}, {})
```

### SQL Injection
```python
# ❌ FORBIDDEN - String formatting
query = f"SELECT * FROM users WHERE id = {user_id}"

# ✅ CORRECT - Parameterization
cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))

# ✅ CORRECT - SQLAlchemy ORM
from sqlalchemy import select
stmt = select(User).where(User.id == user_id)

# ✅ CORRECT - Django ORM
User.objects.filter(id=user_id)
```

---

## 3. Sensitive Data Protection

### Secrets Management
```python
# ❌ FORBIDDEN - Hardcoding secrets
API_KEY = "sk_live_1234567890abcdef"

# ✅ CORRECT - Environment variables
import os
from functools import lru_cache

@lru_cache()
def get_api_key() -> str:
    key = os.environ.get("API_KEY")
    if not key:
        raise ValueError("API_KEY not set")
    return key

# ✅ CORRECT - Django settings
from django.conf import settings
api_key = settings.API_KEY  # Read from environment

# ✅ CORRECT - python-dotenv for development
from dotenv import load_dotenv
load_dotenv()
API_KEY = os.getenv("API_KEY")
```

### Safe Logging
```python
# ❌ FORBIDDEN - Logging sensitive data
logger.info(f"User login: {email}, password: {password}")

# ✅ CORRECT - Sanitize before logging
import copy

def sanitize_sensitive(data: dict) -> dict:
    """Removes sensitive fields for logging."""
    SENSITIVE_FIELDS = {'password', 'token', 'secret', 'api_key', 'ssn'}
    sanitized = copy.deepcopy(data)
    for key in sanitized:
        if any(field in key.lower() for field in SENSITIVE_FIELDS):
            sanitized[key] = '***REDACTED***'
    return sanitized

logger.info(f"User login: {sanitize_sensitive(user_data)}")
```

---

## 4. File Validation

### Safe Upload
```python
import magic
import mimetypes
from pathlib import Path

ALLOWED_EXTENSIONS = {'.pdf', '.jpg', '.jpeg', '.png'}
ALLOWED_MIME_TYPES = {'application/pdf', 'image/jpeg', 'image/png'}
MAX_FILE_SIZE = 10 * 1024 * 1024  # 10MB

def validate_upload(file_path: Path, declared_name: str) -> None:
    # 1. Validate extension
    ext = Path(declared_name).suffix.lower()
    if ext not in ALLOWED_EXTENSIONS:
        raise ValidationError(f"Extension {ext} not allowed")

    # 2. Validate size
    if file_path.stat().st_size > MAX_FILE_SIZE:
        raise ValidationError("File too large")

    # 3. Validate MIME type with python-magic (magic bytes)
    detected = magic.from_file(str(file_path), mime=True)
    if detected not in ALLOWED_MIME_TYPES:
        raise ValidationError(f"File type {detected} not allowed")

    # 4. Validate that extension matches content
    expected_ext = mimetypes.guess_extension(detected)
    if expected_ext and ext != expected_ext:
        raise ValidationError("Extension does not match file content")
```

---

## 5. Web Protection (Django/Flask/FastAPI)

### Django
```python
# ✅ Security settings that should NEVER be removed
# settings.py

SECURE_SSL_REDIRECT = True          # HTTPS required
SECURE_HSTS_SECONDS = 31536000      # HSTS 1 year
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_CONTENT_TYPE_NOSNIFF = True
SECURE_BROWSER_XSS_FILTER = True
SESSION_COOKIE_SECURE = True        # HTTPS only
CSRF_COOKIE_SECURE = True           # HTTPS only
X_FRAME_OPTIONS = 'DENY'            # No clickjacking

# CSRF Protection - NEVER disable
from django.views.decorators.csrf import csrf_exempt  # ⚠️ Only for APIs that need it

# Rate limiting with django-ratelimit
from ratelimit.decorators import ratelimit

@ratelimit(key='ip', rate='5/m', method='POST')
def login_view(request):
    pass
```

### Flask
```python
# ✅ Essential security extensions
from flask import Flask
from flask_talisman import Talisman  # HTTPS, HSTS, CSP
from flask_limiter import Limiter     # Rate limiting
from flask_wtf.csrf import CSRFProtect

app = Flask(__name__)

# NEVER remove these protections
Talisman(app, force_https=True)
CSRFProtect(app)

limiter = Limiter(
    app=app,
    key_func=lambda: request.remote_addr,
    default_limits=["200 per day", "50 per hour"]
)

@app.route('/login', methods=['POST'])
@limiter.limit("5 per minute")  # Rate limiting login
@limiter.limit("1 per second")  # Prevents fast brute force
def login():
    pass
```

### FastAPI
```python
from fastapi import FastAPI, Request
from fastapi.middleware.trustedhost import TrustedHostMiddleware
from fastapi.middleware.cors import CORSMiddleware
from slowapi import Limiter
from slowapi.util import get_remote_address

app = FastAPI()

# NEVER use permissive CORS in production
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://app.example.com"],  # Specific, NOT ['*']
    allow_credentials=True,
    allow_methods=["GET", "POST"],
    allow_headers=["Authorization"],
)

# Rate limiting
limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter

@app.post("/login")
@limiter.limit("5/minute")
async def login(request: Request):
    pass
```

---

## 6. Cryptography

### Password Hashing
```python
# ✅ CORRECT - bcrypt with appropriate salt
import bcrypt

def hash_password(password: str) -> bytes:
    salt = bcrypt.gensalt(rounds=12)  # Cost factor 12
    return bcrypt.hashpw(password.encode(), salt)

def verify_password(password: str, hashed: bytes) -> bool:
    return bcrypt.checkpw(password.encode(), hashed)

# ✅ Alternative - Argon2 (more modern)
from argon2 import PasswordHasher

ph = PasswordHasher(
    time_cost=2,      # Iterations
    memory_cost=65536, # 64MB
    parallelism=1
)

def hash_password(password: str) -> str:
    return ph.hash(password)

def verify_password(password: str, hash_str: str) -> bool:
    try:
        ph.verify(hash_str, password)
        return True
    except:
        return False
```

### JWT Tokens
```python
# ✅ CORRECT - PyJWT with strict verification
import jwt
from datetime import datetime, timedelta

def create_token(user_id: str, secret: str) -> str:
    payload = {
        'user_id': user_id,
        'exp': datetime.utcnow() + timedelta(hours=24),
        'iat': datetime.utcnow(),
        'type': 'access'
    }
    return jwt.encode(payload, secret, algorithm='HS256')

def verify_token(token: str, secret: str) -> dict:
    try:
        return jwt.decode(
            token,
            secret,
            algorithms=['HS256'],
            options={
                'require': ['exp', 'iat'],
                'verify_exp': True,
                'verify_iat': True,
            }
        )
    except jwt.ExpiredSignatureError:
        raise AuthenticationError("Token expired")
    except jwt.InvalidTokenError:
        raise AuthenticationError("Invalid token")
```

---

## 7. Safe Subprocess

```python
# ❌ FORBIDDEN - Shell=True with user input
import subprocess
cmd = f"grep {user_input} file.txt"
subprocess.run(cmd, shell=True)  # Command injection

# ✅ CORRECT - List of arguments, no shell
import shlex
subprocess.run(['grep', user_input, 'file.txt'])

# ✅ STRICT validation if necessary
import re
ALLOWED_PATTERN = re.compile(r'^[a-zA-Z0-9_-]+$')

def safe_grep(pattern: str, filename: str) -> str:
    if not ALLOWED_PATTERN.match(pattern):
        raise ValueError("Invalid pattern")
    result = subprocess.run(
        ['grep', pattern, filename],
        capture_output=True,
        text=True,
        timeout=30  # Mandatory timeout
    )
    return result.stdout
```

---

## 8. Safe XML

```python
# ❌ FORBIDDEN - xml.etree with external data (XXE)
import xml.etree.ElementTree as ET
ET.parse(untrusted_xml)  # Vulnerable to XXE

# ✅ CORRECT - defusedxml
from defusedxml import ElementTree as ET
ET.parse(untrusted_xml)  # Protected against XXE

# ✅ Alternative - lxml with resolvers disabled
from lxml import etree
parser = etree.XMLParser(resolve_entities=False)
tree = etree.parse(untrusted_xml, parser)
```

---

## 9. Context Managers for Resources

```python
# ✅ CORRECT - Always use context managers
def process_file(filename: str) -> None:
    # Guarantees closing even if exception occurs
    with open(filename, 'r') as f:
        data = f.read()

    # Same for connections
    with psycopg2.connect(DATABASE_URL) as conn:
        with conn.cursor() as cur:
            cur.execute("SELECT * FROM users")

    # And for locks
    with threading.Lock():
        update_shared_resource()
```

---

## Python-Specific Checklist

### Security
- [ ] Do not use `pickle`/`yaml.load()`/`eval()` with external data
- [ ] Use `defusedxml` instead of `xml.etree`
- [ ] SQL parameterized with `%s` placeholders
- [ ] Secrets in environment variables, NOT hardcoded
- [ ] Sanitized logging (no passwords/tokens)
- [ ] Upload validation with python-magic
- [ ] CSRF protected in web frameworks
- [ ] HTTPS forced in production
- [ ] Rate limiting on sensitive endpoints
- [ ] bcrypt/argon2 for passwords
- [ ] Subprocess without shell=True
- [ ] JWT with strict claims verification

### Resources
- [ ] Context managers (`with`) for files/connections
- [ ] Timeouts on all I/O operations
- [ ] Size limits on uploads
- [ ] Graceful shutdown with signal handlers

---

## References

- [OWASP Python Security](https://owasp.org/www-project-python-security/)
- [Bandit - Security linter](https://bandit.readthedocs.io/)
- [Safety - Check dependencies](https://pyup.io/safety/)
- [Django Security](https://docs.djangoproject.com/en/stable/topics/security/)
