---
name: linus-kiss-dry-yagni
description: Refactoriza código existente complejo. Usa esta skill cuando el usuario pida simplificar, reescribir, limpiar o mejorar código con funciones largas, código duplicado, anidamiento profundo, parámetros excesivos, lógica críptica tipo clever, dependencias inyectadas o sobre-ingeniería. Aplica KISS, DRY, YAGNI y filosofía Linus Torvalds.
---

# KISS-DRY-YAGNI + Linus Torvalds

Skill directiva de simplicidad. Aplica correcciones automáticamente.

## Quick Reference

| Situación | Recurso |
|-----------|---------|
| Aplicar correcciones | Usa tablas abajo |
| Anti-patrones comunes | `resources/anti-patterns/common.md` |
| Límites y métricas | `resources/decision/metrics.md` |
| Diseñar desde cero | `resources/design/kiss-driven-design.md` |
| Ejemplos Python | `resources/examples/python.md` |
| Ejemplos TypeScript | `resources/examples/typescript.md` |
| Ejemplos Go | `resources/examples/go.md` |
| Ejemplos Kotlin | `resources/examples/kotlin.md` |
| Ejemplos Rust | `resources/examples/rust.md` |
| Decisión arquitectónica | `resources/decision/framework.md` |
| Principios Linus | `resources/principles/linus-torvalds.md` |
| Case studies | `resources/cases/case-studies.md` |
| Estrategias simplificación | `resources/strategies/simplification-strategies.md` |
| Fundamentos | `resources/principles/fundamentals.md` |

## Límites Concretos (NO Negociables)

Estos son los límites máximos. Si los excedes, debes refactorizar:

| Elemento | Límite Máximo | Qué hacer si se excede |
|----------|---------------|------------------------|
| **Líneas por función** | 20 | Dividir en funciones más pequeñas |
| **Parámetros por función** | 4 | Usar objeto/struct para agrupar |
| **Niveles de anidamiento** | 2 | Usar guard clauses |
| **Clases por archivo** | 1 | Separar en archivos |
| **Responsabilidades por clase** | 1 | Extraer responsabilidades |
| **Duplicación de código** | 0 veces | Extraer función común |
| **Interfaces sin uso** | 0 | Eliminar hasta necesitar |
| **Comentarios "qué"** | 0 | Renombrar código |

## Anti-Patrones Comunes

| Anti-Patrón | Señales de Alerta | Solución Inmediata |
|-------------|-------------------|-------------------|
| **Pyramid of Doom** | 3+ niveles de if/else | Guard clauses con returns tempranos |
| **God Function** | Función hace 3+ cosas | Dividir en funciones de 1 responsabilidad |
| **Parameter Explosion** | 5+ parámetros | Objeto/struct config |
| **Interface Pollution** | Interface con 1 implementación | Eliminar interface, usar clase directa |
| **Abstraction Addiction** | Fábricas de fábricas | Simplificar a funciones directas |
| **Future Proofing** | Código "por si acaso" | Eliminar, agregar solo cuando se necesite |
| **Clever Code** | One-liners crípticos | Expandir a código obvio y legible |
| **Comment Cancer** | Comentarios explicando "qué" | Renombrar variables/funciones |

## Reglas de Aplicación Automática

### KISS - Simplificar

| Si detectas | Aplicar |
|------------|---------|
| Función hace 2+ cosas | Dividir en funciones separadas |
| 4+ niveles de anidamiento | Extraer función o usar guard clauses |
| 5+ parámetros | Usar objeto/config struct |
| Nombre necesita comentario | Renombrar |
| Existe solución más simple | Usarla |
| One-liner críptico | Expandir a múltiples líneas claras |
| Lógica compleja sin tests | Simplificar primero, testear después |

### DRY - Centralizar

| Si detectas | Aplicar |
|------------|---------|
| Código idéntico 2+ veces | Extraer función |
| Constantes repetidas | Centralizar |
| Validaciones similares | Crear utilidad común |
| Estructura de datos duplicada | Extraer tipo/interfaz |

**Advertencia**: Código que PARECE igual pero representa conceptos distintos → mantener separado.

### YAGNI - Eliminar

| Si detectas | Aplicar |
|------------|---------|
| Feature "por si acaso" | Eliminar |
| Abstracción sin uso actual | Eliminar |
| Interface con 1 implementación | Eliminar |
| Configuración no requerida | Eliminar |
| Código comentado | Eliminar (está en git) |
| Métodos no usados | Eliminar |
| Dependencias no usadas | Eliminar de imports/requires |

### Orden de Prioridad

1. **YAGNI** → Eliminar primero
2. **KISS** → Simplificar segundo
3. **DRY** → Centralizar tercero

Este orden evita crear abstracciones sobre código que debería eliminarse.

### Excepción: Archivos Esenciales de Proyecto

Los archivos de configuración y estructura de proyecto necesarios para que el código compile/ejecute **NO son YAGNI**. Al crear proyectos nuevos, incluir siempre los archivos esenciales que el lenguaje/framework requiere para funcionar.

## Casos de Estudio Rápidos

### Caso 1: Función God
**Antes**: 80 líneas, valida, calcula, guarda, notifica
**Después**: 4 funciones de 15 líneas cada una
**Por qué**: Cada función hace una cosa, testeable, reusable

### Caso 2: Anidamiento Profundo
**Antes**: 6 niveles de if → imposible de seguir
**Después**: Guard clauses planas → flujo lineal
**Por qué**: El código debe ser legible de arriba a abajo

### Caso 3: Sobre-ingeniería
**Antes**: Interface + Factory + Strategy para 2 opciones
**Después**: If/else o diccionario de funciones
**Por qué**: La simplicidad vence a la "elegancia" teórica

### Caso 4: Duplicación
**Antes**: Validación de email en 5 lugares diferentes
**Después**: Función `isValidEmail()` usada en los 5 lugares
**Por qué**: Un cambio en un solo lugar

## Formato de Salida

**IMPORTANTE**: Siempre ESCRIBIR los archivos al sistema. No solo mostrar código.

```
### Archivos Creados
- path/to/archivo.go (descripción breve)

### Correcciones Aplicadas
- [KISS] Descripción del cambio
- [YAGNI] Descripción del cambio
- [DRY] Descripción del cambio
- [LINUS] Descripción del cambio
```

**Sin preguntas. Sin confirmaciones. Escribir archivos directamente.**

## Checklist Antes de Entregar

- [ ] Funciones tienen ≤20 líneas
- [ ] Máximo 4 parámetros por función
- [ ] Máximo 2 niveles de anidamiento
- [ ] Sin código duplicado
- [ ] Sin interfaces con 1 implementación
- [ ] Sin código "por si acaso"
- [ ] Nombres describen el "qué", no necesitan comentarios
- [ ] Código legible sin explicaciones adicionales

## Cómo Usar

`/kiss-dry-yagni [código o descripción]`

Aplica automáticamente cuando:
- Usuario pide refactorizar código
- Usuario pide simplificar código
- Usuario pide limpiar código
- Usuario pide revisar código
- Hay código con múltiples problemas obvios
- Se detecta sobre-ingeniería
- Se detecta código duplicado

NO aplica cuando:
- Usuario pide explícitamente prototipo/throwaway
- Usuario deshabilita la skill
