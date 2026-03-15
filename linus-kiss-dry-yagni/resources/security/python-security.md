# Seguridad Específica - Python

> Guía de protecciones de seguridad para código Python. NUNCA eliminar estos patrones al aplicar KISS-DRY-YAGNI.

---

## 1. Deserialización Segura

### El Problema: pickle es peligroso
```python
# ❌ PROHIBIDO - NUNCA usar pickle con datos no confiables
import pickle

def process_request(data):
    obj = pickle.loads(data)  # RCE si data es malicioso
    return obj
```

### La Solución
```python
# ✅ CORRECTO - Usar JSON para datos no confiables
import json

def process_request(data: bytes) -> dict:
    return json.loads(data)  # Seguro, no ejecuta código

# ✅ Si necesitas pickle (datos internos), validar origen
def process_internal_data(data: bytes, source: str) -> Any:
    if source not in TRUSTED_INTERNAL_SERVICES:
        raise SecurityError("Untrusted source")
    return pickle.loads(data)
```

### Alternativas Seguras
```python
# JSON - Para datos del cliente/API
import json

# MessagePack - Binario pero seguro
import msgpack

# Protobuf - Estricto y eficiente
from google.protobuf import message

# YAML seguro - NUNCA usar load(), solo safe_load()
import yaml
data = yaml.safe_load(stream)  # ✅
data = yaml.load(stream)       # ❌ Inseguro por defecto
```

---

## 2. Inyección de Código

### eval() y exec()
```python
# ❌ PROHIBIDO - NUNCA usar con input del usuario
def calculate(expression: str) -> Any:
    return eval(expression)  # RCE total

# ✅ CORRECTO - Usar safe_eval o bibliotecas especializadas
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
# ❌ PROHIBIDO - Formato de strings
query = f"SELECT * FROM users WHERE id = {user_id}"

# ✅ CORRECTO - Parametrización
cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))

# ✅ CORRECTO - SQLAlchemy ORM
from sqlalchemy import select
stmt = select(User).where(User.id == user_id)

# ✅ CORRECTO - Django ORM
User.objects.filter(id=user_id)
```

---

## 3. Protección de Datos Sensibles

### Secrets Management
```python
# ❌ PROHIBIDO - Hardcodear secrets
API_KEY = "sk_live_1234567890abcdef"

# ✅ CORRECTO - Variables de entorno
import os
from functools import lru_cache

@lru_cache()
def get_api_key() -> str:
    key = os.environ.get("API_KEY")
    if not key:
        raise ValueError("API_KEY not set")
    return key

# ✅ CORRECTO - Django settings
from django.conf import settings
api_key = settings.API_KEY  # Leído de environment

# ✅ CORRECTO - python-dotenv para desarrollo
from dotenv import load_dotenv
load_dotenv()
API_KEY = os.getenv("API_KEY")
```

### Logging Seguro
```python
# ❌ PROHIBIDO - Loggear datos sensibles
logger.info(f"User login: {email}, password: {password}")

# ✅ CORRECTO - Sanitizar antes de loggear
import copy

def sanitize_sensitive(data: dict) -> dict:
    """Remueve campos sensibles para logging."""
    SENSITIVE_FIELDS = {'password', 'token', 'secret', 'api_key', 'ssn'}
    sanitized = copy.deepcopy(data)
    for key in sanitized:
        if any(field in key.lower() for field in SENSITIVE_FIELDS):
            sanitized[key] = '***REDACTED***'
    return sanitized

logger.info(f"User login: {sanitize_sensitive(user_data)}")
```

---

## 4. Validación de Archivos

### Upload Seguro
```python
import magic
import mimetypes
from pathlib import Path

ALLOWED_EXTENSIONS = {'.pdf', '.jpg', '.jpeg', '.png'}
ALLOWED_MIME_TYPES = {'application/pdf', 'image/jpeg', 'image/png'}
MAX_FILE_SIZE = 10 * 1024 * 1024  # 10MB

def validate_upload(file_path: Path, declared_name: str) -> None:
    # 1. Validar extensión
    ext = Path(declared_name).suffix.lower()
    if ext not in ALLOWED_EXTENSIONS:
        raise ValidationError(f"Extension {ext} not allowed")

    # 2. Validar tamaño
    if file_path.stat().st_size > MAX_FILE_SIZE:
        raise ValidationError("File too large")

    # 3. Validar MIME type con python-magic (magic bytes)
    detected = magic.from_file(str(file_path), mime=True)
    if detected not in ALLOWED_MIME_TYPES:
        raise ValidationError(f"File type {detected} not allowed")

    # 4. Validar que extensión coincide con contenido
    expected_ext = mimetypes.guess_extension(detected)
    if expected_ext and ext != expected_ext:
        raise ValidationError("Extension does not match file content")
```

---

## 5. Protección Web (Django/Flask/FastAPI)

### Django
```python
# ✅ Configuraciones de seguridad que NUNCA eliminar
# settings.py

SECURE_SSL_REDIRECT = True          # HTTPS obligatorio
SECURE_HSTS_SECONDS = 31536000      # HSTS 1 año
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_CONTENT_TYPE_NOSNIFF = True
SECURE_BROWSER_XSS_FILTER = True
SESSION_COOKIE_SECURE = True        # Solo HTTPS
CSRF_COOKIE_SECURE = True           # Solo HTTPS
X_FRAME_OPTIONS = 'DENY'            # No clickjacking

# CSRF Protection - NUNCA desactivar
from django.views.decorators.csrf import csrf_exempt  # ⚠️ Solo para APIs que lo necesiten

# Rate limiting con django-ratelimit
from ratelimit.decorators import ratelimit

@ratelimit(key='ip', rate='5/m', method='POST')
def login_view(request):
    pass
```

### Flask
```python
# ✅ Extensiones de seguridad esenciales
from flask import Flask
from flask_talisman import Talisman  # HTTPS, HSTS, CSP
from flask_limiter import Limiter     # Rate limiting
from flask_wtf.csrf import CSRFProtect

app = Flask(__name__)

# NUNCA eliminar estas protecciones
Talisman(app, force_https=True)
CSRFProtect(app)

limiter = Limiter(
    app=app,
    key_func=lambda: request.remote_addr,
    default_limits=["200 per day", "50 per hour"]
)

@app.route('/login', methods=['POST'])
@limiter.limit("5 per minute")  # Rate limiting login
@limiter.limit("1 per second")  # Evita brute force rápido
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

# NUNCA usar CORS permissivo en producción
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://app.example.com"],  # Específico, NO ['*']
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

## 6. Cryptografía

### Password Hashing
```python
# ✅ CORRECTO - bcrypt con salt apropiado
import bcrypt

def hash_password(password: str) -> bytes:
    salt = bcrypt.gensalt(rounds=12)  # Cost factor 12
    return bcrypt.hashpw(password.encode(), salt)

def verify_password(password: str, hashed: bytes) -> bool:
    return bcrypt.checkpw(password.encode(), hashed)

# ✅ Alternativa - Argon2 (más moderno)
from argon2 import PasswordHasher

ph = PasswordHasher(
    time_cost=2,      # Iteraciones
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

### Tokens JWT
```python
# ✅ CORRECTO - PyJWT con verificación estricta
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

## 7. Subprocess Seguro

```python
# ❌ PROHIBIDO - Shell=True con input del usuario
import subprocess
cmd = f"grep {user_input} file.txt"
subprocess.run(cmd, shell=True)  # Command injection

# ✅ CORRECTO - Lista de argumentos, sin shell
import shlex
subprocess.run(['grep', user_input, 'file.txt'])

# ✅ VALIDACIÓN estricta si es necesario
import re
ALLOWED_PATTERN = re.compile(r'^[a-zA-Z0-9_-]+$')

def safe_grep(pattern: str, filename: str) -> str:
    if not ALLOWED_PATTERN.match(pattern):
        raise ValueError("Invalid pattern")
    result = subprocess.run(
        ['grep', pattern, filename],
        capture_output=True,
        text=True,
        timeout=30  # Timeout obligatorio
    )
    return result.stdout
```

---

## 8. XML Seguro

```python
# ❌ PROHIBIDO - xml.etree con datos externos (XXE)
import xml.etree.ElementTree as ET
ET.parse(untrusted_xml)  # Vulnerable a XXE

# ✅ CORRECTO - defusedxml
from defusedxml import ElementTree as ET
ET.parse(untrusted_xml)  # Protegido contra XXE

# ✅ Alternativa - lxml con resolvers desactivados
from lxml import etree
parser = etree.XMLParser(resolve_entities=False)
tree = etree.parse(untrusted_xml, parser)
```

---

## 9. Context Managers para Recursos

```python
# ✅ CORRECTO - Siempre usar context managers
def process_file(filename: str) -> None:
    # Garantiza cierre incluso si hay excepción
    with open(filename, 'r') as f:
        data = f.read()

    # Lo mismo para conexiones
    with psycopg2.connect(DATABASE_URL) as conn:
        with conn.cursor() as cur:
            cur.execute("SELECT * FROM users")

    # Y para locks
    with threading.Lock():
        update_shared_resource()
```

---

## Checklist Python Específico

### Seguridad
- [ ] No usar `pickle`/`yaml.load()`/`eval()` con datos externos
- [ ] Usar `defusedxml` en lugar de `xml.etree`
- [ ] SQL parametrizado con `%s` placeholders
- [ ] Secrets en variables de entorno, NO hardcodeados
- [ ] Logging sanitizado (no passwords/tokens)
- [ ] Validación de uploads con python-magic
- [ ] CSRF protegido en frameworks web
- [ ] HTTPS forzado en producción
- [ ] Rate limiting en endpoints sensibles
- [ ] bcrypt/argon2 para passwords
- [ ] Subprocess sin shell=True
- [ ] JWT con verificación estricta de claims

### Recursos
- [ ] Context managers (`with`) para archivos/conexiones
- [ ] Timeouts en todas las operaciones I/O
- [ ] Límites de tamaño en uploads
- [ ] Graceful shutdown con signal handlers

---

## Referencias

- [OWASP Python Security](https://owasp.org/www-project-python-security/)
- [Bandit - Security linter](https://bandit.readthedocs.io/)
- [Safety - Check dependencies](https://pyup.io/safety/)
- [Django Security](https://docs.djangoproject.com/en/stable/topics/security/)
