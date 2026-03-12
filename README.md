# KISS-DRY-YAGNI + Linus Torvalds

[![Skills](https://skills.sh/badge.svg)](https://skills.sh)

> **Skill de refactorización de código** aplicando principios de simplicidad: KISS (Keep It Simple, Stupid), DRY (Don't Repeat Yourself), YAGNI (You Ain't Gonna Need It) y la filosofía de Linus Torvalds.

## Instalación

```bash
npx skills add millerfrankmc/skills/linus-kiss-dry-yagni
```

Soportado en: Claude Code, Cursor, Codex, OpenCode, y [más agentes](https://skills.sh).

---

## Qué hace esta skill

Esta skill transforma código complejo y sobre-ingenierizado en código simple, directo y mantenible.

### Problemas que resuelve

| Problema | Solución |
|----------|----------|
| Funciones de 100+ líneas | Dividir en funciones ≤20 líneas (≤50 para seguridad) |
| Anidamiento de 6+ niveles | Guard clauses planas |
| 10+ parámetros en funciones | Objeto/struct config |
| Interfaces con 1 implementación | Eliminar abstracción innecesaria (excepto seguridad) |
| Código duplicado | Extraer función común |
| Fábricas de fábricas | Simplificar a funciones directas |
| Código "por si acaso" | Eliminar (YAGNI) - excepto defensa en profundidad |
| Over-engineering de seguridad | **Preservar** validaciones, headers, cifrado |

### 🔒 Seguridad Integrada

Esta skill **no sacrifica seguridad por simplicidad**:

- ✅ Preserva validaciones de entrada (múltiples capas)
- ✅ Mantiene SQL parametrizado (nunca convierte a interpolación)
- ✅ Conserva headers de seguridad HTTP
- ✅ Respeta funciones de seguridad (bcrypt, JWT, CSRF)
- ✅ Mantiene comentarios críticos (`// SECURITY:`, `// NUNCA`)
- ✅ No aplica DRY a validaciones con diferentes threat models

> **"La simplicidad nunca debe sacrificar la seguridad."**

### Resultados medidos

En pruebas A/B contra baseline sin skill:

| Métrica | Mejora |
|---------|--------|
| Pass rate | **+43%** (100% vs 57%) |
| Líneas de código | **-42%** (más simple) |
| Tiempo de refactorización | **-9.6%** |

### 🔐 Benchmark de Seguridad

8 evaluaciones exhaustivas verifican que la skill **preserva protecciones críticas**:

| Categoría | Evaluaciones | Pass Rate |
|-----------|--------------|-----------|
| Simplificación correcta | God Function, JWT Validation | 100% ✅ |
| Preservación de seguridad | Auth, SQL Injection, Input Validation | 100% ✅ |
| Inteligencia contextual | DRY vs Context, Headers, Comments | 100% ✅ |
| **Total verificaciones de seguridad** | **40/40** | **100%** |

> Ver `/skill-creator-workspace/benchmark-results.md` para detalles completos.

---

## Uso

Una vez instalada, la skill se activa automáticamente cuando:

- Pides "refactorizar código"
- Pides "simplificar código"
- Pides "limpiar código"
- Pides "revisar código"
- Se detecta código con múltiples problemas obvios

### Ejemplo de uso

**Input:**
```python
def process_user(user_id, user_name, user_email, user_phone, user_address,
                 user_city, user_country, is_active, is_premium, created_at):
    if user_id:
        if user_name:
            if user_email:
                if '@' in user_email and '.' in user_email:
                    if is_active:
                        # ... 50 líneas más de anidamiento
```

**Output (automático):**
```python
def process_user(user: User) -> Optional[dict]:
    if not user.id: return None
    if not user.name: return None
    if not is_valid_email(user.email): return None
    if not user.is_active: return None

    discount = 0.20 if user.is_premium else 0.10
    return build_user_data(user, discount)
```

---

## Principios Aplicados

### KISS - Keep It Simple, Stupid
- Funciones pequeñas con una sola responsabilidad
- Código que se lee de arriba a abajo
- Sin clever code (one-liners crípticos)

### DRY - Don't Repeat Yourself
- Extraer código duplicado
- Centralizar validaciones
- Un cambio en un solo lugar

### YAGNI - You Ain't Gonna Need It
- Eliminar código "por si acaso" (excepto defensa en profundidad de seguridad)
- Sin abstracciones prematuras
- Sin interfaces con una sola implementación (excepto auth/crypto)

### Filosofía Linus Torvalds
> "The way to write clean code is to not write clever code."

- Guard clauses con returns tempranos
- Mensajes de error específicos
- Código obvio sobre código elegante

---

## Límites Concretos (No Negociables)

| Elemento | Límite | Acción si se excede |
|----------|--------|---------------------|
| **Líneas por función** | 20 (50 para seguridad) | Dividir en funciones más pequeñas |
| **Parámetros por función** | 4 | Usar objeto/struct |
| **Niveles de anidamiento** | 2 | Usar guard clauses |
| **Duplicación de código** | 0 | Extraer función común |
| **Interfaces sin uso** | 0 | Eliminar hasta necesitar |

---

## Recursos Incluidos

- **Anti-patrones comunes** - Identificación y soluciones
- **Protección de seguridad** - Guardias que preservan validaciones y controles de seguridad
- **Ejemplos por lenguaje** - Python, TypeScript, Go, Kotlin, Rust
- **Case studies** - Refactorizaciones reales
- **Estrategias de simplificación** - Técnicas específicas
- **Principios de Linus Torvalds** - Filosofía detrás del código simple

---

## Anti-Patrones Detectados

| Anti-Patrón | Señal de Alerta | Solución |
|-------------|-----------------|----------|
| **Pyramid of Doom** | 3+ niveles de if/else | Guard clauses |
| **God Function** | Función hace 3+ cosas | Dividir en funciones |
| **Parameter Explosion** | 5+ parámetros | Objeto config |
| **Interface Pollution** | Interface con 1 implementación | Usar clase directa |
| **Abstraction Addiction** | Fábricas de fábricas | Funciones directas |
| **Future Proofing** | Código "por si acaso" | Eliminar |
| **Clever Code** | One-liners crípticos | Expandir a código claro |

---

## Casos de Estudio

### Caso 1: Función God (Python)
- **Antes:** 80 líneas, valida, calcula, guarda, notifica
- **Después:** 4 funciones de 15 líneas cada una
- **Beneficio:** Testeable, reusable, mantenible

### Caso 2: Anidamiento Profundo (Go)
- **Antes:** 7 niveles de if → imposible de seguir
- **Después:** Guard clauses planas → flujo lineal
- **Beneficio:** Legible de arriba a abajo

### Caso 3: Sobre-ingeniería (TypeScript)
- **Antes:** Interface + Factory + Strategy para 2 opciones
- **Después:** If/else simple
- **Beneficio:** 75% menos código, más claro

---

## Estructura del Repositorio

```
.
├── LICENSE                     # MIT License
├── README.md                   # Este archivo
└── linus-kiss-dry-yagni/       # Skill principal
    ├── SKILL.md                # Definición de la skill
    ├── evals/
    │   └── evals.json          # Casos de prueba A/B
    └── resources/
        ├── anti-patterns/      # Anti-patrones comunes
        ├── cases/              # Casos de estudio
        ├── decision/           # Frameworks de decisión
        ├── design/             # Diseño guiado por KISS
        ├── examples/           # Ejemplos por lenguaje
        ├── principles/         # Principios fundamentales
        └── strategies/         # Estrategias de simplificación
```

> **Nota:** Este repositorio `skills` contiene múltiples skills. Cada skill está en su propia carpeta.

---

## Testing

### A/B Testing - Simplificación

- **5 casos de prueba** (Python, TypeScript, Go, Kotlin, Ruby)
- **100% pass rate** con la skill vs 57% sin skill
- **42% reducción** en líneas de código promedio

### Benchmark de Seguridad

- **8 evaluaciones exhaustivas** verifican preservación de seguridad
- **40 verificaciones de seguridad** - 100% pass rate
- **0 simplificaciones inseguras** detectadas

Ver `/skill-creator-workspace/benchmark-results.md` para resultados detallados.

---

## Contribuir

¿Encontraste un caso donde la skill no funciona bien? Abre un issue con:

1. Código de entrada
2. Lo que esperabas
3. Lo que obtuviste

---

## Licencia

MIT

---

## Inspiración

> "Talk is cheap. Show me the code."
> — Linus Torvalds

Esta skill existe porque el código simple vence al código complejo. No más over-engineering. No más abstracciones innecesarias. Solo código que funciona y se puede mantener.
