# Métricas y Límites de Complejidad

Estos son los límites concretos que guían las decisiones de refactorización.

## Límites NO Negociables

Si se exceden estos límites, DEBES refactorizar:

| Elemento | Límite Máximo | Acción al Exceder |
|----------|---------------|-------------------|
| **Líneas por función** | 20 | Dividir en funciones más pequeñas |
| **Parámetros por función** | 4 | Usar objeto/struct config |
| **Niveles de anidamiento** | 2 | Usar guard clauses |
| **Clases por archivo** | 1 | Separar en archivos |
| **Responsabilidades por función** | 1 | Extraer funciones |
| **Responsabilidades por clase** | 1 | Dividir en clases |
| **Duplicación de código** | 0 | Extraer función común |
| **Interfaces sin uso** | 0 | Eliminar hasta necesitar |
| **Comentarios explicando "qué"** | 0 | Renombrar código |
| **Código no usado** | 0 | Eliminar (está en git) |

## Umbrales de Complejidad (Referencia)

| Métrica | Verde ✅ | Amarillo ⚠️ | Rojo ❌ |
|---------|----------|-------------|---------|
| **Líneas por función** | 1-20 | 21-50 | 50+ |
| **Parámetros por función** | 1-3 | 4 | 5+ |
| **Niveles de anidamiento** | 1-2 | 3 | 4+ |
| **Complejidad ciclomática** | 1-5 | 6-10 | 10+ |
| **Dependencias por archivo** | 1-5 | 6-10 | 10+ |
| **Funciones por clase** | 1-10 | 11-20 | 20+ |
| **Métodos públicos por clase** | 1-5 | 6-10 | 10+ |
| **Longitud de nombre** | 5-30 chars | 31-50 | 50+ |

## Cómo Medir

### Líneas de Código

```bash
# Python - contar líneas de función
wc -l archivo.py

# JavaScript/TypeScript
npx cloc archivo.ts

# Go
gocloc archivo.go
```

### Complejidad Ciclomática

```bash
# Python
radon cc archivo.py -a

# JavaScript
npx complexity-report archivo.js

# Go
gocyclo archivo.go
```

### Niveles de Anidamiento

Contar manualmente los niveles máximos de:
- `if` / `else` / `elif`
- `for` / `while`
- `try` / `catch` / `finally`
- `switch` / `case`
- Funciones anidadas

### Parámetros

```bash
# Contar en Python
grep -E "^def \w+\(" archivo.py

# Contar en TypeScript
grep -E "function \w+\(|\w+\(.*:" archivo.ts
```

## Reglas de Decisión

### Cuándo Extraer una Función

Extraer función si:
- [ ] Tiene >20 líneas
- [ ] Hace 2+ cosas diferentes
- [ ] Tiene comentario explicando una sección
- [ ] Tiene código que podría reusarse
- [ ] Es difícil de nombrar claramente

### Cuándo Usar un Objeto Config

Usar objeto/struct si:
- [ ] Función tiene >4 parámetros
- [ ] Parámetros son opcionales
- [ ] Parámetros pertenecen a un concepto
- [ ] Se pasan juntos frecuentemente

### Cuándo Eliminar una Interface

Eliminar interface si:
- [ ] Tiene solo 1 implementación
- [ ] No hay planes de segunda implementación
- [ ] Agrega indirección sin valor

### Cuándo Aplicar Guard Clauses

Aplicar guard clauses si:
- [ ] Hay >2 niveles de anidamiento
- [ ] La función retorna temprano en casos de error
- [ ] Hay validaciones al inicio que anidan el código real

## Ejemplos de Cálculo

### Ejemplo 1: Función con 25 líneas

```python
def process_user(data):  # Línea 1
    if not data:         # Línea 2
        return None      # Línea 3

    validated = validate(data)   # Línea 4
    if not validated:            # Línea 5
        return None              # Línea 6

    transformed = transform(validated)   # Línea 7
    saved = save(transformed)            # Línea 8
    notified = notify(saved)             # Línea 9
    logged = log(notified)               # Línea 10

    return logged        # Línea 11
```

**Total**: 11 líneas (✅ Dentro del límite)

### Ejemplo 2: Función con 5 parámetros

```python
# ❌ ANTES: 5 parámetros
def create_user(name, email, phone, address, is_active):
    pass

# ✅ DESPUÉS: 1 objeto
def create_user(data: UserData):
    pass
```

### Ejemplo 3: Anidamiento de 4 niveles

```python
# ❌ ANTES: 4 niveles
def process(order):
    if order:                    # Nivel 1
        if order.valid:          # Nivel 2
            if order.payment:    # Nivel 3
                if order.paid:   # Nivel 4
                    fulfill(order)

# ✅ DESPUÉS: 1 nivel
def process(order):
    if not order: return
    if not order.valid: return
    if not order.payment: return
    if not order.paid: return
    fulfill(order)
```

## Anti-Patrones de Métricas

### No optimices métricas por métricas

**Mal:**
```python
# Dividir función solo para tener menos líneas
def process_part1():
    # 10 líneas
    pass

def process_part2():
    # 10 líneas
    pass

def process_part3():
    # 10 líneas
    pass
```

**Bien:**
```python
# Dividir por responsabilidad
def validate():
    pass

def transform():
    pass

def save():
    pass
```

### No ignores el contexto

Algunas funciones legítimamente necesitan más líneas:
- Mappers de configuración
- Validadores complejos
- Algoritmos matemáticos

Pero deben ser la excepción, no la regla.

## Métricas de Calidad Final

Después de aplicar KISS-DRY-YAGNI, el código debe:

- [ ] Cumplir todos los límites de complejidad
- [ ] Ser entendible sin explicaciones
- [ ] Ser testeable sin mocks complejos
- [ ] Ser modificable sin miedo
- [ ] Tener <5% de duplicación de código
- [ ] Tener 0 interfaces con 1 implementación
- [ ] Tener 0 código no usado
