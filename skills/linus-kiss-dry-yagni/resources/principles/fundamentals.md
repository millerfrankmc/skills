# Fundamentos KISS-DRY-YAGNI

Este recurso profundiza en los principios fundamentales para quien quiere entenderlos mejor.

## KISS - Keep It Simple, Stupid

### Origen
Principio diseñado por la Marina de EE.UU. en 1960. La simplicidad debe ser el objetivo principal.

### Por qué importa
- Código simple = menos bugs
- Código simple = más fácil de mantener
- Código simple = más fácil de entender
- Código simple = más rápido de desarrollar

### La trampa
"Simple" no significa "primitivo". Significa:
- Fácil de entender
- Fácil de modificar
- Fácil de testear
- Hace una cosa bien

## DRY - Don't Repeat Yourself

### Origen
Acuñado por Andy Hunt y Dave Thomas en "The Pragmatic Programmer" (1999).

### La regla
"Every piece of knowledge must have a single, unambiguous, authoritative representation within a system."

### Distinción clave
- **DRY**: Mismo concepto, mismo código → centralizar
- **No DRY**: Código similar, conceptos diferentes → mantener separado

### Ejemplo de distinción
```python
# Mismo concepto - DRY apropiado
def validate_email(email): ...
def validate_user_email(user): return validate_email(user.email)
def validate_admin_email(admin): return validate_email(admin.email)

# Conceptos diferentes - NO aplicar DRY
def calculate_user_tax(user): ...  # Lógica de impuestos usuarios
def calculate_product_tax(product): ...  # Lógica de impuestos productos
# Aunque parecen similares, son dominios diferentes
```

## YAGNI - You Aren't Gonna Need It

### Origen
Principio de Extreme Programming (XP). Atribuido a Ron Jeffries.

### La filosofía
"Always implement things when you actually need them, never when you just foresee that you need them."

### Por qué cuesta
- Los desarrolladores queremos "hacerlo bien desde el principio"
- Tememos que "después va a ser más difícil"
- Nos gusta la arquitectura elegante

### La realidad
- 80% de las features "por si acaso" nunca se usan
- Agregar flexibilidad tiene costo ahora
- Cuando necesitas la flexibilidad, ya no es la forma correcta

### Test de necesidad
Antes de agregar algo, pregúntate:
1. ¿Lo necesito HOY?
2. ¿Hay un requisito explícito?
3. ¿Hay un usuario esperando esto?
4. ¿El dolor de no tenerlo es real y actual?

Si todas son "no" → YAGNI, no lo implementes.
